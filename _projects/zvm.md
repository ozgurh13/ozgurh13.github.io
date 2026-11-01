---
layout: page
title: zvm
description:
  shared-nothing, work-stealing, fiber-concurrent, threaded M:N scheduler
---

###### Accurate Up-to-Commit: [e19bbd4][commit]


A shared-nothing, fiber-concurrent, stack-based threaded interpreter featuring
io_uring, work-stealing M:N scheduler, mark & sweep GC, loadable plugins and
FFI.

Definitions (w.r.t. zvm):

 - `shared-nothing`: Each fiber gets their own isolated stack and heap and doesn't share any (mutable) state.

 - `fiber-concurrent`: The main abstraction for concurrency is a lightweight, user-space thread/coroutine. Cheap to create, cheap to context switch.

 - `stack-based`: Execution is done using a stack machine. Other models include register machines and graph reduction.

 - `threaded`: Instead of using a giant switch case, instruction dispatch is done using labels and direct jumps to the label.

 - `work-stealing`: If a scheduler runs out of fibers to execute, it actively tries to "steal" fibers from other schedulers.

 - `M:N scheduler`: M fibers are scheduled on N (OS) threads.


Dependencies:
 - `libffi 3.7.1`: FFI
 - `liburing 2.15`: io_uring
 - `lightning 2.2.3`: JIT compilation
 - `mimalloc 3.4.3`: memory allocation


Architecture
```
    +------------------------------------------------------------------------+
    |                                   zvm                                  |
    |                  controls schedulers, manages fiber creation           |
    +------------------------------------------------------------------------+
                 |                       |                       |
              pthread                 pthread                 pthread
                 |                       |                       |
      +--------------------+  +--------------------+  +--------------------+
      |      scheduler     |  |      scheduler     |  |      scheduler     |
      |  pinned to core 0  |  |  pinned to core 1  |  |  pinned to core N  |
      +--------------------+  +--------------------+  +--------------------+
      | - io_uring ring    |  | - io_uring ring    |  | - io_uring ring    |
      | - fiber queue      |  | - fiber queue      |  | - fiber queue      |
      +--------------------+  +--------------------+  +--------------------+



    +------------------+
    |      fiber       |
    +------------------+
    | - native stack   |
    | - mi_heap_t heap |
    | - execution ctx  |
    +------------------+

```


---

## Core structures

The core data structures used in the vm.


### zvm_t

This is the core structure. It represents the virtual machine.

{% highlight c %}
struct zvm {
	pthread_t *scheduler_ids;
	zvm_scheduler_t **schedulers;
	size_t nschedulers;

	struct {
		size_t active;
		mi_heap_t *heap;
		zvm_fiber_id_t next_id;
	} fiber;

	pthread_t plugin_manager_tid;
	zvm_plugin_manager_t plugin_manager;
};
{% endhighlight %}


`nschedulers` is the number of schedulers, `schedulers` are a list of pointers, `scheduler_ids` is a list of pthread ids.

`fiber.active` stores the current number of active fibers,
`fiber.heap` is the heap the fibers are allocated on and
`fiber.next_id` stores the *next* fiber's id.
All created fibers have unique ids, and are not reused when the fiber dies.

`plugin_manager_tid` is the plugin manager's thread id and `plugin_manager` is the plugin manager.



### zvm_scheduler_t

The scheduler represents the abstraction that fibers run on.

{% highlight c %}
struct zvm_scheduler {
	zvm_scheduler_id_t id;

	atomic_bool active;

	struct io_uring ring;
	int eventfd;

	struct {
		zvm_fiber_t *fiber;
		void *sp;
	} current;

	ringbuffer_t *fibers;

	pcg32_random_t pcg;

	zvm_t *zvm;
};
{% endhighlight %}

`id` is the scheduler's core id (the core on which it is pinned). Only one scheduler can be pinned to a given core, and does not migrate to another core during the lifetime of the vm.

`active` is the current state of the scheduler. `true` means running and `false` mean exited.
Once a scheduler transitions into the exited state, it cannot transition back into the running state.

`ring` is the io_uring instance for this scheduler. Like the scheduler, it is pinned to the same core and optimized for single issuer and polling from the kernel.
`eventfd` is the eventfd for this scheduler, used to signal it asynchronously.

`current.fiber` is the fiber executing on the scheduler, `current.sp` is the stack pointer.

`fibers` is a FIFO queue of fibers this scheduler will run.
It currently uses a mutex-backed ringbuffer of fixed size, which is not ideal.

[`pcg`][pcg] is the random number generator used.

`zvm` is the virtual machine.


### zvm_fiber_t

The fiber contains a native stack and a virtual heap, and executes instructions.

{% highlight c %}
struct zvm_fiber {
	zvm_fiber_id_t id;

	zvm_fiber_state_t state;
	zvm_fiber_flags_t flags;

	struct {
		void *sp;
		void *stack;
		zvm_heap_t heap;
	} memory;

	zvm_execution_t *execution;
	size_t reductions;

	zvm_t *zvm;
	zvm_scheduler_t *scheduler;
};
{% endhighlight %}

`id` is the fiber's id, is unique and not recycled.

`state` is the current state the fiber is in, and is defined as 
{% highlight c %}
typedef enum {
	/*  ready to execute; can be placed on any scheduler  */
	ZVM_FIBER_STATE_READY,

	/*  currently executing on a scheduler  */
	ZVM_FIBER_STATE_RUNNING,

	/*  blocked on I/O, should NOT be stolen  */
	ZVM_FIBER_STATE_BLOCKED_IO,

	/*  blocked on z3, should NOT be stolen  */
	ZVM_FIBER_STATE_BLOCKED_Z3,

	/*  execution halted, fiber should be deallocated  */
	ZVM_FIBER_STATE_DEAD,
} zvm_fiber_state_t;
{% endhighlight %}
`ZVM_FIBER_STATE_BLOCKED_Z3` is currently not used.

`flags` is defined but currently not used and will contain the fiber's properties.

The `memory` struct is the fiber's memory;
`sp` is the stack pointer and `stack` and `heap` are self documenting.
Here the stack is the actual stack that the CPU will use when running the fiber.
On x86_64 it needs to be 16 byte aligned.

`execution` contains the execution context (instructions and virtual stack).

`reductions` is how many reductions (in total) the fiber has executed so far.

`zvm` is the vm and `scheduler` is the scheduler that the fiber is running on.


### zvm_execution_t

This is the execution context.

{% highlight c %}
struct zvm_execution {
	const zvm_instruction_t *instructions;
	size_t ip;

	zvm_value_t stack[ZVM_EXECUTION_STACK_SIZE];
	size_t sp;

	zvm_symbol_table_t symbol_table;

	zvm_execution_prompt_t prompts[ZVM_EXECUTION_PROMPT_STACK_SIZE];
	size_t prompt_index;
};
{% endhighlight %}

`instructions` is the list of instructions to execute, `ip` is the instruction pointer.

`stack` here refers to the *stack-based*ness of the vm.
When executing instructions values are pushed to and popped from the stack. `sp` is the stack pointer. 
If later on we wanted to move to a *register-based* vm, we would only need to swap this out with a version that uses registers.

`symbol_table` stores the interned strings, is created once and is read-only.
It uses robin hood hashing for fast lookups and GNU obstack for arena allocation.

`prompts` and `prompt_index` are used for [delimited continuations][delimited-continuations].


### zvm_value_t

A value is anything that can be pushed on the stack.

{% highlight c %}
struct zvm_value {
	zvm_value_type_t type;
	union {
		uint64_t u64;

		ffi_type *ffi_type;
		void *ffi_value;
		void *ffi_fnptr;

		zvm_object_t *object;
	} as;
};
{% endhighlight %}

Currently very limited, you can only push an unsigned 64 bit number or an object.
FFI is pretty much hard-coded.

This type is important, it allows to push both boxed and unboxed values on the stack uniformly.


### zvm_object_t

Objects are boxed values, and are subject to garbage collection.

{% highlight c %}
struct zvm_object {
	struct {
		bool marked;
		zvm_object_t *next;
	} gc;

	zvm_object_type_t type;
	union {
		uint64_t u64;

		zvm_object_ffi_function_t ffi_function;

		zvm_object_continuation_t continuation;
	} as;
};
{% endhighlight %}

Again, pretty minimal. A uint64_t, an FFI function or a continuation.
These are allocated on the fiber's heap.



### zvm_heap_t

A heap is where fibers allocate objects. They also track extra info regarding GC.

{% highlight c %}
struct zvm_heap {
	mi_heap_t *heap;

	struct {
		zvm_object_t* stack[ZVM_HEAP_GC_NOGC_STACK_SIZE];
		size_t sp;
	} nogc;

	zvm_object_t *objects;

	uint64_t gc_threshold;

	size_t object_count;
};
{% endhighlight %}

`nogc` is a safe haven for objects that are unreachable, but shouldn't be gc'd.
This turns out to be especially useful when we need to pop objects off the stack and create new objects, which might end up triggering GC.

`objects` contains all allocated objects, `object_count` contains current number of allocated objects and `gc_threshold` is when GC will be triggered.



### zvm_instruction_t & zvm_opcode_t

{% highlight c %}
typedef enum {
	/*  no op  */
	ZVM_OPCODE_NOP,

	/*  halt execution (of fiber)  */
	ZVM_OPCODE_HALT,

	/*  check reduction  */
	ZVM_OPCODE_REDUCTION,


	/*  test io_uring integration  */
	ZVM_OPCODE_TEST_IO,

	/*  test push add print  */
	ZVM_OPCODE_TEST_PUSH,
	ZVM_OPCODE_TEST_POP,
	ZVM_OPCODE_TEST_MAKE_UINT64,      // create a heap allocated uint64 object
	ZVM_OPCODE_TEST_ADD,
	ZVM_OPCODE_TEST_PRINT,

	/*  test ffi  */
	ZVM_OPCODE_TEST_FFI_PUSH_TYPE,             /*  when creating the function  */
	ZVM_OPCODE_TEST_FFI_PUSH_FNPTR,            /*  when creating the function  */
	ZVM_OPCODE_TEST_FFI_PUSH_VALUE,            /*  when calling the function */
	ZVM_OPCODE_TEST_FFI_MAKE_FUNCTION,         /*  takes arg count (not including return type)  */
	ZVM_OPCODE_TEST_FFI_CALL,                  /*  call function on top of the stack  */

	/*  test continuations  */
	ZVM_OPCODE_TEST_RESET,
	ZVM_OPCODE_TEST_SHIFT,
	ZVM_OPCODE_TEST_RESUME,

	/*  should be last  */
	_ZVM_OPCODE_COUNT,
} zvm_opcode_t;


struct zvm_instruction {
	zvm_opcode_t opcode;
	zvm_value_t arg;
};
{% endhighlight %}

This is executed in the execution context.


---

## Design & Implementation

Misc info
 - The vm is first compiled into a static library, then a wrapper program is used to create a driver.
 - `*_create` functions allocate memory, `*_init` initialize structures. `*_destroy` is used to deallocate, `*_free` is used to de-initialize.
 - All allocation is done using [mimalloc][mimalloc].
 - Error handling is minimal.
 - Design is kept simple.
   This is to get a working version up and running before moving on to a more complex/intricate design.


### Overall


To create a zvm instance:
{% highlight c %}
static void
zvm_init(zvm_t *zvm)
{
	zvm->fiber.active  = 0;
	zvm->fiber.heap    = mi_heap_new();
	zvm->fiber.next_id = 0;

	/* ... */
}

zvm_t *
zvm_create(size_t nschedulers)
{
	zvm_t *zvm = mi_malloc(sizeof(zvm_t));
	assert (zvm != NULL);

	zvm_init(zvm);

	zvm_schedulers_start(zvm, nschedulers);

	return zvm;
}
{% endhighlight %}
`nschedulers` is the number of schedulers to start.
This function will probably be changed to take a zvm_config_t.

This will create the zvm instance and start the schedulers.
Each scheduler will be spawned on a separate pthread.


To create a scheduler:
{% highlight c %}
static void
zvm_scheduler_init(zvm_scheduler_t *scheduler, zvm_scheduler_id_t id)
{
	struct io_uring_params params = {0};
	params.flags = /* ... */
	params.sq_thread_cpu = id;

	int ret;
	if ((ret = io_uring_queue_init_params(QUEUE_DEPTH, &scheduler->ring, &params)) < 0) {
		/* ... */
	}

	scheduler->eventfd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);
	if (scheduler->eventfd < 0) {
		/* ... */
	}
	zvm_scheduler_arm_notify(scheduler);

	scheduler->fibers = ringbuffer_create(1024 * 128);

	/* ... */
}

zvm_scheduler_t *
zvm_scheduler_create(zvm_t *zvm, zvm_scheduler_id_t id)
{
	zvm_scheduler_t *scheduler = mi_calloc(1, sizeof(zvm_scheduler_t));
	if (scheduler == NULL) {
		return NULL;
	}

	scheduler->id = id;
	scheduler->zvm = zvm;

	scheduler->active = false;

	zvm_scheduler_init(scheduler, id);

	return scheduler;
}
{% endhighlight %}

Here the `id` is the core id. Because no two schedulers share the same core, the core's id is used to identify the scheduler.

This will initialize the io_uring instance for the core, register its eventfd (using io_uring) and seed the random number generator.
It will *not* pin the scheduler to the core or start the scheduler's loop.

To start the scheduler:
{% highlight c %}
void*
zvm_scheduler_main(void *arg)
{
	zvm_scheduler_t *scheduler = arg;
	zvm_scheduler_pin_to_core(scheduler);
	scheduler->active = true;

	while (scheduler->active) {
		zvm_fiber_t *fiber = zvm_scheduler_get_next_runnable_fiber(scheduler);
		if (fiber == NULL) {
			/*  here we don't have any work to do  */
			zvm_scheduler_wait_cqe(scheduler);
			continue;
		}

		assert(fiber != NULL);
		zvm_scheduler_yield_to_fiber(fiber);

		switch (fiber->state) {
			case ZVM_FIBER_STATE_READY:
				zvm_scheduler_reschedule_fiber(scheduler, fiber);
				break;

			case ZVM_FIBER_STATE_DEAD:
				zvm_fiber_destroy(fiber);
				break;

			case ZVM_FIBER_STATE_RUNNING:
				/*  unreachable state!  */
				assert(false);

			case ZVM_FIBER_STATE_BLOCKED_IO:
				break;
		}
	}

	zvm_scheduler_cleanup(scheduler);

	return NULL;
}
{% endhighlight %}

This is the function passed to pthread_create, with the scheduler being the arg.
This function will pin the scheduler to the core, set the scheduler's `active` flag to true and enter the main loop.

The main loop consists of the following steps:
  1. Grab the next runnable fiber (`zvm_scheduler_get_next_runnable_fiber`)
     1. Flush io_uring's completion queue.
        This will push any fibers that have completed I/O back into the queue.
     2. Check its own fiber queue, if a fiber is present return it
     3. Try to steal from other schedulers. If successful, return a fiber.
     4. If all else fails wait for short period of time on io_uring, then GOTO 1.
  2. Yield to the fiber, wait for it to yield back (`zvm_scheduler_yield_to_fiber`)
  3. Act upon the fiber's status
     - `READY`: reschedule it
     - `DEAD`: deallocate it
     - `RUNNING`: abort, this should never happen
     - `BLOCKED_IO`: do nothing, the fiber is submitted to io_uring (as user_data) and will be reaped off the completion queue.



Context switching is just saving registers and swapping stack pointers:
{% highlight nasm %}
.intel_syntax noprefix

.global zvm_fiber_switch
.global zvm_fiber_entry_stub

# void zvm_fiber_switch(void** old_sp, void* new_sp);
zvm_fiber_switch:
    # 1. save caller/callee-saved registers of current fiber onto its stack
    push rbp
    push rbx
    push r12
    push r13
    push r14
    push r15

    # 2. save current stack pointer to *old_sp
    mov QWORD PTR [rdi], rsp

    # 3. switch stack pointer to new_sp
    mov rsp, rsi

    # 4. restore saved registers of new fiber from its stack
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx
    pop rbp

    # 5. jump to return address (pushed on new stack)
    ret


# entry point wrapper for new fibers
zvm_fiber_entry_stub:
    # arguments passed in registers per x86_64 ABI:
    #     r12 = fiber pointer (self)
    #     r13 = entry function pointer
    mov rdi, r12
    call r13

    # if the entry function returns, fiber is dead. crash or handle gracefully.
    ud2
{% endhighlight %}
This uses the fiber's stack to save the registers.
Another option would be to define `typedef struct { uint64_t rbp, rsp, r12, r13, r14, r15 } registers_t` and use that.

Context switching is handled using a few helper functions:
{% highlight c %}
static void
zvm_scheduler_yield_to_fiber_impl(zvm_fiber_t *fiber)
{
	zvm_scheduler_t *scheduler = fiber->scheduler;

	zvm_fiber_t *prev = scheduler->current.fiber;
	scheduler->current.fiber = fiber;

	void **sp = prev ? &prev->memory.sp : &scheduler->current.sp;
	zvm_fiber_switch(sp, fiber->memory.sp);
}

static void
zvm_scheduler_yield_to_scheduler_impl(zvm_fiber_t *fiber)
{
	zvm_scheduler_t *scheduler = fiber->scheduler;

	zvm_fiber_t* prev = scheduler->current.fiber;
	scheduler->current.fiber = NULL;

	zvm_fiber_switch(&prev->memory.sp, scheduler->current.sp);
}

void
zvm_scheduler_yield_to_fiber(zvm_fiber_t *fiber)
{
	fiber->state = ZVM_FIBER_STATE_RUNNING;
	zvm_scheduler_yield_to_fiber_impl(fiber);
}

void
zvm_scheduler_yield_to_scheduler(zvm_fiber_t *fiber)
{
	/*
	 * only if the fiber was running, it may have terminated
	 */
	if (fiber->state == ZVM_FIBER_STATE_RUNNING) {
		fiber->state = ZVM_FIBER_STATE_READY;
	}
	zvm_scheduler_yield_to_scheduler_impl(fiber);
}

void
zvm_scheduler_block_on_io(zvm_fiber_t *fiber)
{
	fiber->state = ZVM_FIBER_STATE_BLOCKED_IO;
	zvm_scheduler_yield_to_scheduler_impl(fiber);
}
{% endhighlight %}


Fibers are created using an execution context.
There are some assumptions made about the instructions being executed:
 1. `HALT` is used to halt the execution.
 2. `REDUCTION` is placed where needed (function/loop boundaries, etc.)

Fiber creation is dynamic, they can be created from anywhere at any time.

{% highlight c %}
zvm_fiber_t *
zvm_fiber_create(zvm_t *zvm, zvm_execution_t *execution)
{
	zvm_fiber_t *fiber = mi_heap_malloc(zvm->fiber.heap, sizeof(zvm_fiber_t));
	if (fiber == NULL) {
		/* ... */
	}

	fiber->id = zvm_get_next_fiber_id(zvm);
	zvm_fiber_count_increment(zvm);

	zvm_fiber_init(fiber);

	fiber->execution = execution;

	/*  stack should be 16 byte aligned, per x86_64 ABI  */
	fiber->memory.stack = mi_heap_malloc_aligned(
		zvm->fiber.heap,
		ZVM_FIBER_STACK_SIZE,
		ZVM_FIBER_STACK_ALIGNMENT
	);
	zvm_fiber_stack_setup(fiber, zvm_fiber_entry);

	fiber->state = ZVM_FIBER_STATE_READY;
	fiber->flags = ZVM_FIBER_FLAG_INITIAL;

	fiber->zvm = zvm;

	fiber->reductions = 0;

	return fiber;
}
{% endhighlight %}


When a fiber is run, it enters the execution loop:
{% highlight c %}
void
zvm_fiber_entry(zvm_fiber_t *fiber)
{
	zvm_execution_t *execution = fiber->execution;

	static const reduction_t budget = /* ... */;
	reduction_t spent = 0;

	for (;;) {
		if (fiber->state == ZVM_FIBER_STATE_DEAD) {
			zvm_scheduler_yield_to_scheduler(fiber);
			break;
		}

		reduction_t reductions = zvm_execution_step(execution, fiber);
		fiber->reductions += reductions;
		spent += reductions;

		if (spent >= budget) {
			zvm_scheduler_yield_to_scheduler(fiber);
			spent = 0;
		}
	}
}
{% endhighlight %}

Contrary to its name, `zvm_execution_step` doesn't execute a single instruction, it runs until it hits a REDUCTION instruction.
It is also responsible for setting the state of the fiber to `DEAD` if execution halts.

{% highlight c %}
reduction_t
zvm_execution_step(zvm_execution_t *execution, zvm_fiber_t *fiber)
{
	reduction_t reductions = 0;

	static const void* dispatch[] = {
		[ZVM_OPCODE_NOP]                      = &&do_ZVM_OPCODE_NOP,
		[ZVM_OPCODE_HALT]                     = &&do_ZVM_OPCODE_HALT,
		[ZVM_OPCODE_REDUCTION]                = &&do_ZVM_OPCODE_REDUCTION,

		/* ... */
	};
	static_assert(
		(sizeof(dispatch) / sizeof(*dispatch)) == _ZVM_OPCODE_COUNT,
		"dispatch table is missing opcodes/labels"
	);


	zvm_instruction_t current;
#define  DISPATCH()                                                                      \
do {                                                                                     \
    current = zvm_execution_get_next_instruction(execution);                             \
    zvm_opcode_t opcode = current.opcode;                                                \
    reductions += opcode_to_reduction[opcode];                                           \
    goto *dispatch[opcode];                                                              \
} while (0)

	DISPATCH();

do_ZVM_OPCODE_NOP: {
    DISPATCH();
}  /* do_ZVM_OPCODE_NOP */

do_ZVM_OPCODE_HALT: {
    fiber->state = ZVM_FIBER_STATE_DEAD;
    return reductions;
}  /* do_ZVM_OPCODE_HALT */

do_ZVM_OPCODE_REDUCTION: {
    return reductions;
}  /* do_ZVM_OPCODE_REDUCTION  */

	/* ... */

#undef  DISPATCH
}
{% endhighlight %}

A few important notes:
 1. `static_assert((sizeof(dispatch) / sizeof(*dispatch)) == _ZVM_OPCODE_COUNT)` doesn't trigger if you add an instruction in the middle of the enumeration, only if you add it right before _ZVM_OPCODE_COUNT. C will happily assume that anything that isn't explicitly initialized is implicitly initialized :)
 2. `opcode_to_reduction` is an estimate as to how much an instruction should cost. Might be wildly off.

The `DISPATCH` macro and `[ZVM_OPCODE_NOP] = &&do_ZVM_OPCODE_NOP` is the *threaded* part of the interpreter. `dispatch` is a table of computed gotos and DISPATCH handles the machinery to directly jump to the next label in the instruction sequence.

I didn't really focus on making the language, so execution only has a value stack.

Currently instructions are used to test features:

 - io_uring
{% highlight c %}
do_ZVM_OPCODE_TEST_IO: {
    /*
     * 1. create a buffer to read data
     * 2. submit sqe entry (set data as ourselves)
     * 3. yield_to_scheduler as blocked on IO
     *    this is important as the scheduler shouldn't put us back on the fibers queue, another scheduler might steal us and that would be disastrous
     * 4. when scheduler gets cqe, it will reschedule us and we continue executing from here
     *    at this point, it's ok if we get stolen
     */

    {
        int fd = open(/* ... */);
        char buffer[4096] = {0};
        struct io_uring *ring = &fiber->scheduler->ring;
        struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
        io_uring_prep_read(sqe, fd, buffer, 4096, 0);
        io_uring_sqe_set_data(sqe, fiber);
        io_uring_submit(ring);

        /*  need to wait for I/O to complete  */
        zvm_scheduler_block_on_io(fiber);
        /*  I/O is done here, resume execution  */

        printf("%.*s\n", 10, buffer);
        close(fd);
    }

    DISPATCH();
}  /* do_ZVM_OPCODE_TEST_IO */
{% endhighlight %}

It is imperative that the fiber isn't stolen during this time.
The submission is tied to the scheduler's ring, and will be reaped from there.


 - delimited continuation
{% highlight c %}
do_ZVM_OPCODE_TEST_RESET: {
    size_t sp = execution->sp;
    execution->prompts[execution->prompt_index++] = (zvm_execution_prompt_t) {
        .sp = sp,
    };

    DISPATCH();
}  /* do_ZVM_OPCODE_TEST_RESET */

do_ZVM_OPCODE_TEST_SHIFT: {
    zvm_execution_prompt_t prompt = execution->prompts[--execution->prompt_index];
    size_t sp = execution->sp;
    size_t frame_count = sp - prompt.sp - 1;
    zvm_object_t *k = zvm_heap_object_malloc_continuation(fiber, execution->ip, frame_count);
    memcpy(k->as.continuation.frames, &execution->stack[sp], frame_count * sizeof(zvm_value_t));
    execution->sp -= frame_count;
    execution->stack[execution->sp++] = (zvm_value_t) {
        .type      = ZVM_VALUE_TYPE_OBJECT,
        .as.object = k,
    };

    DISPATCH();
}  /* do_ZVM_OPCODE_TEST_SHIFT */

do_ZVM_OPCODE_TEST_RESUME: {
    zvm_value_t value = execution->stack[--execution->sp];
    zvm_object_t *k = value.as.object;
    memcpy(&execution->stack[execution->sp], k->as.continuation.frames, k->as.continuation.frame_count * sizeof(zvm_value_t));
    execution->sp += k->as.continuation.frame_count;
    execution->ip = k->as.continuation.pc + 1;

    DISPATCH();
}  /* do_ZVM_OPCODE_TEST_RESUME */
{% endhighlight %}

This only copies the stack values.


 - FFI
{% highlight c %}
do_ZVM_OPCODE_TEST_FFI_PUSH_TYPE: {
    zvm_value_t arg = current.arg;
    execution->stack[execution->sp++] = arg;
    DISPATCH();
}  /* do_ZVM_OPCODE_TEST_FFI_PUSH_TYPE */

do_ZVM_OPCODE_TEST_FFI_PUSH_FNPTR: {
    zvm_value_t arg = current.arg;
    execution->stack[execution->sp++] = arg;
    DISPATCH();
}  /* do_ZVM_OPCODE_TEST_FFI_PUSH_FNPTR */

do_ZVM_OPCODE_TEST_FFI_MAKE_FUNCTION: {
    zvm_value_t arg = current.arg;
    size_t cffi_arity = arg.as.u64;

    /*  function ptr  */
    zvm_value_t value0 = execution->stack[--execution->sp];
    void *fnptr = value0.as.ffi_fnptr;

    zvm_object_t *fn = zvm_heap_object_malloc_ffi_function(fiber, fnptr, cffi_arity);

    /*  return type  */
    zvm_value_t value = execution->stack[--execution->sp];
    ffi_type *rettype = value.as.ffi_type;

    ffi_type *argtypes[cffi_arity];
    for (uint64_t i = 0; i < cffi_arity; i++) {
        zvm_value_t value = execution->stack[--execution->sp];
        argtypes[i] = value.as.ffi_type;
    }

    ffi_status status = ffi_prep_cif(&fn->as.ffi_function.cif, FFI_DEFAULT_ABI, cffi_arity, rettype, argtypes);
    switch (status) {
        case FFI_OK:
            break;

        case FFI_BAD_ARGTYPE:
            fprintf(stderr, "FFI_BAD_ARGTYPE\n");
            break;

        case FFI_BAD_TYPEDEF:
            fprintf(stderr, "FFI_BAD_TYPEDEF (One or more data types are invalid/malformed).\n");
            break;

        case FFI_BAD_ABI:
            fprintf(stderr, "FFI_BAD_ABI (The requested ABI is unsupported or invalid).\n");
            break;

        default:
            fprintf(stderr, "Unknown libffi error.\n");
            break;
    }

    execution->stack[execution->sp++] = (zvm_value_t) {
        .type      = ZVM_VALUE_TYPE_OBJECT,
        .as.object = fn,
    };

    DISPATCH();
}  /* do_ZVM_OPCODE_TEST_FFI_MAKE_FUNCTION */

do_ZVM_OPCODE_TEST_FFI_PUSH_VALUE: {
    zvm_value_t arg = current.arg;
    execution->stack[execution->sp++] = arg;
    DISPATCH();
}  /* do_ZVM_OPCODE_TEST_FFI_PUSH_VALUE */

do_ZVM_OPCODE_TEST_FFI_CALL: {
    zvm_value_t value = execution->stack[--execution->sp];
    zvm_object_t *fn = value.as.object;

    for (uint64_t i = 0; i < fn->as.ffi_function.arity; i++) {
        zvm_value_t value = execution->stack[--execution->sp];
        fn->as.ffi_function.args[i] = value.as.ffi_value;
    }

    uint64_t result;

    ffi_call(&fn->as.ffi_function.cif, FFI_FN(fn->as.ffi_function.fnptr), &result, fn->as.ffi_function.args);

    execution->stack[execution->sp++] = (zvm_value_t) {
        .type      = ZVM_VALUE_TYPE_OBJECT,
        .as.object = zvm_heap_object_malloc_uint64(fiber, result),
    };

    DISPATCH();
}  /* do_ZVM_OPCODE_TEST_FFI_CALL */
{% endhighlight %}

FFI is hard-coded in the interpreter.

The next step would be to create a `ctypes` equivalent module that handles marshaling between objects and primitive types and handle all of this. FFI shouldn't be a part of the instruction set.

VLAs should also be swapped out for something better.


### Work Stealing

When a scheduler runs out of fibers it can execute (doesn't have any in its queue and io_uring has no completions) it will actively try to steal fibers from other schedulers.
Starting from its neighbors, it will check if the victim has at least `ZVM_SCHEDULER_STEAL_THRESHOLD` fibers available. If so it will steal half of the fibers for itself.

{% highlight c %}
static size_t
zvm_scheduler_try_steal(zvm_scheduler_t *scheduler)
{
	size_t stolen = 0;

	zvm_t *zvm = scheduler->zvm;

	ringbuffer_t *fibers = scheduler->fibers;

	const size_t id = scheduler->id;
	const size_t nschedulers = zvm->nschedulers;
#define  ZVM_SCHEDULER_STEAL_THRESHOLD   /* ...  */
	for (size_t i = 0; i < nschedulers - 1; i++) {
		/*
		 * we want to alternate offset like this
		 *   +1  -1  +2  -2  ...
		 */

		/* ... */

		size_t s_id = /* ... */

		assert(/* s_id >= 0 && */ s_id < nschedulers && s_id != id);

		zvm_scheduler_t *s = zvm->schedulers[s_id];
		assert(s != NULL);

		if (!s->active) {
			continue;
		}

		ringbuffer_t *s_fibers = s->fibers;
		assert(s_fibers != NULL);

		size_t rb_size = ringbuffer_size(s_fibers);
		if (rb_size < ZVM_SCHEDULER_STEAL_THRESHOLD) {
			continue;
		}

		size_t amount = rb_size / 2;
		for (size_t j = 0; j < amount; j++) {
			zvm_fiber_t *fiber = ringbuffer_timedread(s_fibers, 1 /*  millisecond  */);
			if (fiber != NULL) {
				/*  when we steal, set the scheduler to us  */
				fiber->scheduler = scheduler;
				ringbuffer_write(fibers, fiber);
				stolen++;
			}
		}
	}

	return stolen;
}
{% endhighlight %}

The next iteration of the scheduler design is to be NUMA-aware as well as distinguish between P-cores and E-cores.
The main idea is to keep locality (not to arbitrarily cross NUMA boundaries) as well as distribute work according to the core the scheduler is pinned on. Another idea I have is to have P-cores be preemptive instead of cooperative, allowing for less overhead when executing CPU intensive work.

The stealing algorithm, modeled in TLA+:
```
\* can the thief steal from the victim
\*   numa nodes must be the same (cannot steal from other numa nodes)
\*   scheduler types must be the same (P and E cannot cross-steal)
\*   victim must have fibers available
CanStealFrom(thief, victim) ==
  /\ thief.core # victim.core
  /\ thief.numa = victim.numa
  /\ thief.type = victim.type
  /\ Len(fiberQueues[victim.core]) > 0

StealFrom(thiefID, victimID) ==
  LET thief  == Schedulers[thiefID]
      victim == Schedulers[victimID]
  IN /\ CanStealFrom(thief, victim)
     /\ LET fiber          == Head(fiberQueues[victim.core])
            newVictimQueue == Tail(fiberQueues[victim.core])
            newThiefQueue  == Append(fiberQueues[thief.core], fiber)
        IN /\ fiberStates[fiber] = "Ready"
           /\ fiberQueues' = [fiberQueues EXCEPT 
                             ![victim.core] = newVictimQueue,
                             ![thief.core]  = newThiefQueue]
           /\ UNCHANGED <<fiberRunning, fiberStates, fiberReductions, fiberRounds>>
```

### Plugins

The plugin system allows you to load `.so` files dynamically.
When the vm starts, it starts a pthread that listens on a UNIX socket.
There is also another program (`zvm-ctl`) that has a cli interface that handles the communication with the vm.

The protocol is a simple request-response model:
{% highlight c %}
/*  sent from zvm-ctl to zvm  */
typedef struct zvm_plugin_request  zvm_plugin_request_t;
/*  sent from zvm to zvm-ctl  */
typedef struct zvm_plugin_response zvm_plugin_response_t;


typedef enum {
	ZVM_PLUGIN_REQUEST_PLUGIN_LOAD,
	ZVM_PLUGIN_REQUEST_PLUGIN_UNLOAD,
} zvm_plugin_request_type_t;

struct zvm_plugin_request {
	zvm_plugin_api_version_t api_version;

	zvm_plugin_request_type_t type;

	union {
		struct {
			char filepath[PATH_LENGTH];
		} load;

		struct {
			char name[NAME_LENGTH];
		} unload;
	};
} __attribute__((packed));


typedef enum {
	ZVM_PLUGIN_RESPONSE_STATUS_OK,
	ZVM_PLUGIN_RESPONSE_STATUS_ERROR,
} zvm_plugin_response_type_t;

struct zvm_plugin_response {
	zvm_plugin_response_type_t type;

	union {
		char error[ERROR_LENGTH];
	};
} __attribute__((packed));
{% endhighlight %}

Example:
{% highlight bash %}
# while zvm is running ...

# load a plugin
$ zvm-ctl load /path/to/plugin.so
ZVM_PLUGIN_RESPONSE_STATUS_OK

# unload a plugin
$ zvm-ctl unload plugin
ZVM_PLUGIN_RESPONSE_STATUS_OK
{% endhighlight %}

When loading a plugin, pass the full path to the `.so` file.
When unloading a plugin, pass the name of the plugin. (Yes, you can have multiple plugins with the same name, this isn't an error (yet!). When unloading, zvm will unload the first plugin it finds with that name.)

A simple plugin:
{% highlight c %}
/*
 * a simple plugin example
 */
#include <stdio.h>
#include <unistd.h>
#include <stdatomic.h>

#include "zvm/zvm.h"
#include "zvm/plugin.h"

static volatile atomic_bool active = false;

static bool
_init(zvm_t *zvm)
{
	puts("plugin init called");

	return true;
}

static void
_free(zvm_t *zvm)
{
	puts("plugin free called");
}

static void
_start(zvm_t *zvm)
{
	puts("plugin start called");
	active = true;
	while (active) {
		printf("zvm nschedulers: %zu\n", zvm->nschedulers);
		sleep(1);
	}
	puts("plugin work done");
}

static bool
_stop(zvm_t *zvm)
{
	puts("plugin stop called");
	active = false;

	return true;
}


static const zvm_plugin_interface_t plugin_interface = {
	.name  = "plug0",
	.init  = _init,
	.free  = _free,
	.start = _start,
	.stop  = _stop,
};

ZVM_PLUGIN_EXPORT
const zvm_plugin_interface_t *
zvm_load_plugin(void)
{
	return &plugin_interface;
}
{% endhighlight %}

To write a plugin you need to implement `const zvm_plugin_interface_t *zvm_load_plugin(void);`.
This returns a struct of functions:
{% highlight c %}
struct zvm_plugin_interface {
	const char* name;            /*  name of the plugin  */

	zvm_plugin_fn_init_t init;
	zvm_plugin_fn_free_t free;

	zvm_plugin_fn_start_t start;
	zvm_plugin_fn_stop_t  stop;
};
{% endhighlight %}

The lifecycle of the plugin:

  1. `init`: This is where the plugin should initialize its state.
  2. `start`: This is where the plugin begins its work.
  3. `stop`: This is where the plugin should end its work.
  4. `free`: This is where the plugin should free its state.


Caveats:
 - When loading a plugin, the vm's dtrace probes don't trace the plugin. You need to trace the plugin separately.



### Other Notes

`playground/` is a collection of isolated tests for ideas.
 - `intern.c`: string interning
 - `marksweep.c`: mark-and-sweep GC
 - `preemptive.c`: preemptive scheduler
 - `threadcancel.c`: cancel a thread
 - `jitffi.c`: calling JIT'ed function using libffi
 - `trace/`: dtrace probes
 - `plugin/`: loadable plugins


### Known Issues & Future Work

 - Bug fixes & Refactoring:
    1. Creating lots of fibers causes segmentation faults. `zvm_fiber_destroy` seems to be the issue.
 - Improvements:
    1. Scheduler initialization should have a barrier, this way the vm waits until all schedulers are online before proceeding.
    2. Fiber's native stack is allocated alongside itself on zvm's fiber heap. This should be changed to use `mmap` with stack protection.
    3. Fibers can only submit single requests to io_uring, they should also be able to submit multishot requests.
    4. `zvm_execution_t` should have a proper frames and an environment.
    5. NUMA-aware scheduler:
       This should be a runtime feature toggled using options.
       This way we can compare performance.
 - Programming Language


[commit]: https://github.com/ozgurh13/zvm/tree/e19bbd4de3185d81889dc2dff63fd1049b7760e6
[pcg]: https://www.pcg-random.org/
[delimited-continuations]: https://en.wikipedia.org/wiki/Delimited_continuation
[mimalloc]: https://github.com/microsoft/mimalloc
