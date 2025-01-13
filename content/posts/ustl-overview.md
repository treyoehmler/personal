---
title: User Space Thread Library Overview
date: 2025-01-12
description: Overview of user space thread library
summary: sticky
tags:
  - welcome
  - new
  - about
  - first
---
oh so sticky
{add some overview, like 3-4 sentences} - not sure this will end up being a public doc in some way, probably just gonna use this to lay out the various phases and base requirements, tests, etc.

test test test
## Requirements

#### Thread management

- functions to create a new user thread, specifying:
	- function pointer for threads start routine
	- optional or default stack size
- mechanism to exit thread (either return from start or by calling thread_exit())
- thread_join(thread_id) function so one thread can wait for another to finish

#### Scheduling

- maintain ready queue (or multiple if using priorities)
- at minimum, cooperative, round-robin scheduling
- example: `thread_yield()` causes current thrad to save context and place itself at end of ready queue, then resume next thread
- optionally implement preemptive scheduling via time interrupt
	- `SIGALRM` + `setitimer` ensuring limited time slice for each thread

#### Synchronization primitives

- provide at least one sync construct
- mutex (create, lock, unlock)
- semaphore (init, wait, post)
- condition variable (if using more advanced thread_wait(), thread_signal() model)

#### Stack management

- each thread must have own stack - allocate memory via malloc or mmap
- switch between stack pointers and program counters during context switch

#### Thread-Safe I/O

- if choose cooperative approach, single blocking system call eg. read(2) on a fd can block the entire process
- not required to solve fully but document behavior
- might provide non-block or poll-based approach
- extra credit to integrate with scheduler

## Roadmap

#### Phase 1: Basic thread creation + context switching

Write minimal thread struct with:
- context (`ucontext_t` or `jmp` buffer with `setjmp/longjmp`)
- stack pointer (allocated via malloc / mmap)
- thread id / index
- state (`READY`, `RUNNING`, `BLOCKED`, `FINISHED`)

Implement
- `thread_create()`
- `thread_yield()`
- global or static scheduler structure with queue of ready threads

Deliverable: demo program creates a few threads, prints messages in round-robin

#### Phase 2: Thread exit and join

Lets threads finish by either:
- returning from their start routine
- calling `thread_exit()`

Implement:
- `thread_join(thread_id)` so one thread can wait for another to finish, retrieving an exit code (if choose to support)
- stack memory deallocation management for once thread is truly done

Deliverable: program where main thread creates worked threads and uses thread_join() to wait for them to finish

#### Phase 3: Synchronization

Implement synchronization primitives (cv, mutex, semaphore)
- `mutex_init(mutext_t*)`
- `mutex_lock(mutext_t*)`
- `mutex_unlock(mutex_t*)`
Start with mutex implementation and then either semaphore or cv after

Possible store queue or threads waiting on mutex - if thread calls lock while another holds it, blocks cooperatively (not 100% what co-op means in this context)

Deliverable: classic producer/consumer or bounded buffer test (not sure what that is) verifying sync primitives prevent race conditions


#### Phase 4: Preemptive scheduling

• Use a **timer** (setitimer) to generate SIGALRM every X milliseconds.
• The signal handler performs a context switch if a thread’s timeslice is up.
• Mark certain functions (like thread_lock() or your scheduler code) as “signal-safe” or disable interrupts within them as needed.

**Deliverable**: Show that threads still get interleaved if they never yield voluntarily.

----
### [Neco](https://github.com/tidwall/neco?tab=readme-ov-file)

{notes about neco, ie structure, style, implementation details, ect}

---
### Project Objectives

1. **Understanding Context Switching**
    Learn to manage CPU registers, stack pointers, and instruction pointers for each user-level thread. You will use either:
    - The POSIX `getcontext(2)`, `setcontext(2)`, `swapcontext(2)`, and `makecontext(2)` APIs (though these might be considered legacy on some systems), or
    - Inline assembly or compiler intrinsics to save and restore CPU state manually.
2. **Cooperative vs. Preemptive Scheduling**
    - At a minimum, implement _cooperative scheduling_, where threads voluntarily yield control back to the scheduler.
    - **Optional (often required in graduate-level courses):** implement _preemptive scheduling_ by handling signals/interrupts (e.g., using an interval timer with `setitimer(2)`, or `SIGALRM` signal) so that the library can forcibly context-switch threads to ensure fairness.
3. **Scheduling Policies**
    - Implement a straightforward _round-robin_ scheduler or a _priority-based_ scheduler.
    - Maintain ready queues or priority queues for threads.
4. **Thread Lifecycle Management**
    - **Creation** (`thread_create`): allocate a stack, initialize thread control block (TCB), and place the thread in the scheduler’s ready list.
    - **Yielding** (`thread_yield`): cooperative context switch that allows another thread to run.
    - **Exit** (`thread_exit`): gracefully terminate a thread, release resources, and possibly unblock threads waiting on `thread_join`.
    - **Join** (`thread_join`): allow one thread to wait for another thread’s completion and retrieve any return value.
    - **Detaching** (`thread_detach`): indicate that a thread’s resources should be reclaimed automatically upon exit, without a join.
5. **Synchronization**
    Implement user-level synchronization primitives:
    - **Mutexes** (`thread_mutex_init`, `thread_mutex_lock`, `thread_mutex_unlock`, `thread_mutex_destroy`)
    - **Condition Variables** (`thread_cond_init`, `thread_cond_wait`, `thread_cond_signal`, `thread_cond_broadcast`, `thread_cond_destroy`)
    - **Semaphores** (optional but common as an alternative or supplement to mutex/cond): binary or counting semaphores for resource management.
6. **Thread-Local Storage (TLS)**
    - Provide a mechanism (e.g., `thread_key_create`, `thread_setspecific`, `thread_getspecific`, `thread_key_delete`) so each thread can store data that is global to itself but does not collide with other threads.
    - This is particularly useful for storing state specific to each thread (e.g., error codes, buffers).
7. **Signals and Thread Cancellation**
    - **Cancellation**: Provide a means to cancel threads safely (`thread_cancel`). You can choose either asynchronous cancellation (immediate) or deferred cancellation (only at certain cancellation points like `thread_yield`, lock acquisition, etc.).
    - **Signal Handling**: If you implement preemption using signals, handle them carefully so they do not disrupt critical sections.
8. **Robustness and Error Handling**
    - Properly handle error codes (e.g., out-of-memory scenarios) and invalid parameters.
    - Ensure that your library will not leak resources: thread stacks, TCBs, synchronization objects, etc.
9. **Documentation and Usability**
    - Provide a single public header (`uthread.h` or similar) documenting all functions in your library.
    - Write a Makefile or build script that compiles the library and each test program.
    - Include inline comments that explain important design decisions (scheduler, data structures, etc.).
    - Provide **extensive** debugging messages or logs (compile-time configurable) to aid in diagnosing concurrency bugs.

### Architecture

- **Thread Control Block (TCB)**
    A typical TCB contains:
    - Thread ID (integer or pointer)
    - Execution context (register set, stack pointer, program counter)
    - Stack pointer to dynamically allocated memory
    - Thread state (`READY`, `RUNNING`, `BLOCKED`, `EXITED`, etc.)
    - Scheduling information (priority, time slice, etc.)
    - Synchronization info (mutexes held, waiting condition, etc.)
    - Optional: saved signal mask, thread-local storage pointer, cancellation flags
- **Scheduler Data Structures**
    - **Ready Queue**: A queue (FIFO) or priority queue for threads that are _READY_ to run.
    - **Wait Queues**: For threads blocked on mutexes, condition variables, or semaphores.
    - **Zombie/Exited Queue** (optional): Threads that are done running but not yet joined/detached.
- **Context Switch Mechanism**
    - For cooperative scheduling, a call to `thread_yield()` or a blocking call in synchronization triggers `swapcontext(…)` (or equivalent assembly routine).
    - For preemptive scheduling, a `SIGALRM` (or similar) handler forcibly performs a context switch after a time quantum. Must handle re-entrancy carefully.
- **Memory Management**
    - Dynamically allocate stacks (e.g., using `malloc`) for each thread.
    - Ensure stacks are a safe size (e.g., 64 KB to a few MB, depending on memory constraints).
    - Some implementations use a guard page at the end of the stack to catch overflows (platform-dependent).
- **Synchronization**
    - Implement a simple spinlock or disable signals (if preemptive) while modifying shared data structures in your library’s code.
    - Use internal structures for condition variables that queue waiting threads and schedule them when signaled.
- **Error Checking**
    - If `thread_create` fails to allocate memory, return an error code.
    - Validate pointers passed to your API (e.g., mutex pointers).
    - Prevent double locks (if lock reentrancy is not supported).

### Project Requirements

```c
#ifndef UTHREAD_H
#define UTHREAD_H

#include <stddef.h>

/* Thread type / ID. You can define it as a struct or pointer. */
typedef struct uthread *uthread_t;

/* ========== Thread Management ========== */

/* Creates a new thread that starts execution at 'start_routine(arg)'.
   stack_size can be fixed or user-specified.
   Returns a thread ID or NULL on error. */
uthread_t uthread_create(void *(*start_routine)(void *), void *arg);

/* Exits the currently running thread with a given 'retval'. */
void uthread_exit(void *retval);

/* Blocks the calling thread until the thread 'tid' finishes.
   The return value of the finished thread is stored in *retval. */
int uthread_join(uthread_t tid, void **retval);

/* Detaches the thread so it cannot be joined later. Resources
   are automatically freed on exit. */
int uthread_detach(uthread_t tid);

/* Voluntarily yields to another thread (cooperative sched). */
void uthread_yield(void);

/* Initialization function for the threading system.
   Typically sets up main thread context, signal handlers, etc. */
int uthread_init(void);

/* ========== Synchronization Primitives ========== */

typedef struct uthread_mutex uthread_mutex_t;
typedef struct uthread_cond  uthread_cond_t;

/* Mutex API */
int uthread_mutex_init(uthread_mutex_t *mutex);
int uthread_mutex_lock(uthread_mutex_t *mutex);
int uthread_mutex_unlock(uthread_mutex_t *mutex);
int uthread_mutex_destroy(uthread_mutex_t *mutex);

/* Condition variable API */
int uthread_cond_init(uthread_cond_t *cond);
int uthread_cond_wait(uthread_cond_t *cond, uthread_mutex_t *mutex);
int uthread_cond_signal(uthread_cond_t *cond);
int uthread_cond_broadcast(uthread_cond_t *cond);
int uthread_cond_destroy(uthread_cond_t *cond);

/* ========== Thread-Local Storage ========== */

typedef unsigned int uthread_key_t;
int uthread_key_create(uthread_key_t *key, void (*destructor)(void*));
int uthread_setspecific(uthread_key_t key, const void *value);
void *uthread_getspecific(uthread_key_t key);
int uthread_key_delete(uthread_key_t key);

/* ========== Cancellation (Optional) ========== */
int uthread_cancel(uthread_t tid);

#endif /* UTHREAD_H */

```

#### Additional Requirements

- **Preemptive Scheduling**: If required, configure a timer (e.g., `setitimer`) to send `SIGALRM` every X milliseconds. Use a signal handler to call your scheduler.
- **Nested Synchronization**: Make sure that if multiple threads are waiting on a single mutex or condition, the scheduler handles them in a fair manner (round-robin or priority).
- **Performance**: Demonstrate that your user-level scheduling does not significantly degrade the performance of concurrent tasks (compared to naive single-threaded solutions).

### Test Programs

1. validate basic create / join functionality
	- two threads will print their greetings
	- main thread will join both and return their values

```c
#include "uthread.h"
#include <stdio.h>

void *thread_main(void *arg) {
    printf("Hello from thread %d!\n", (int)(size_t)arg);
    return (void*)(size_t)((int)(size_t)arg + 100);
}

int main() {
    uthread_init();

    uthread_t t1 = uthread_create(thread_main, (void*)1);
    uthread_t t2 = uthread_create(thread_main, (void*)2);

    void *retval1, *retval2;
    uthread_join(t1, &retval1);
    uthread_join(t2, &retval2);

    printf("Thread 1 returned %d, Thread 2 returned %d.\n",
           (int)(size_t)retval1, (int)(size_t)retval2);

    return 0;
}
```

2. yield + cooperative scheduling context switching test
	- thread alternate printing message (in purely cooperative manner)

```c
#include "uthread.h"
#include <stdio.h>

void *counter(void *arg) {
    int i;
    for (i = 0; i < 5; i++) {
        printf("[Thread %d] i = %d\n", (int)(size_t)arg, i);
        uthread_yield();
    }
    return NULL;
}

int main() {
    uthread_init();

    uthread_t t1 = uthread_create(counter, (void*)1);
    uthread_t t2 = uthread_create(counter, (void*)2);

    uthread_join(t1, NULL);
    uthread_join(t2, NULL);

    return 0;
}
```


3. mutex + condition variable
4. preemptive scheduling
5. stress test + memory leak detection
	- create / destroy large number of threads
	- ensure library does not leak memory
	- use `valgrind` to verify no leaks
	- sample idea
		- start 1000 threads, each doing trivial work and exiting
		- repeat process 5 times in loop
		- if no leaks or crashes, test passed






