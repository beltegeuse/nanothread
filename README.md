# nanothread — Minimal thread pool for task parallelism

## Introduction

This library provides a minimal cross-platform interface for task parallelism.
Given a computation that is partitioned into a set of interdependent tasks, the
library efficiently distributes this work to a thread pool using lock-free
queues, while respecting dependencies between tasks.

Each task is associated with a callback function that is potentially invoked
multiple times if the task consists of multiple work units. This whole
process is arbitrarily recursive: task callbacks can submit further jobs, wait
for their completion, etc. Parallel loops, reductions, and more complex
graph-based computations are easily realized using these abstractions.

This project is internally implemented in C++11, but exposes the main
functionality using a pure C99 API, along with a header-only C++11 convenience
wrapper. It has no dependencies other than CMake and a C++11-capable compiler.
The entire project requires less than 1000 lines of header and
implementation code (according to [cloc](http://cloc.sourceforge.net/)).

This library is part of the larger
[Dr.Jit](https://github.com/mitsuba-renderer/drjit) project and parallelizes
workloads generated by the
[Dr.Jit-Core](https://github.com/mitsuba-renderer/drjit-core) library. However,
this project has no dependencies on these parent projects and can be used in
any other context.

## Why?

Many of my previous projects have built on [Intel's Thread Building
Blocks](https://software.intel.com/content/www/us/en/develop/tools/threading-building-blocks.html)
for exactly this type of functionality. Unfortunately, large portions of TBB's
task interface were recently deprecated as part of the oneAPI / oneTBB
transition. Rather than struggling with this complex dependency, I decided to
build something minimal and stable that satisfies my requirements.

## Examples (C++11 interface)

The follow examples showcase the C++11 interface, which is a thin header-only
layer over the C99 API.

### Parallel for loops (synchronous)
```cpp
template <typename T, typename Func>
void parallel_for(const blocked_range<T> &range, Func &&func, Pool *pool = nullptr);
```
This function submits a single task consisting of a arbitrarily many work units
that are processed in blocks of a specified size, and waits for their
completion. If no thread pool ``Pool *`` is specified, the default pool will be
used (and created on the fly, if needed).

Example:

```cpp
#include <nanothread/nanothread.h>

namespace nt = nanothread;

int main(int, char **) {
    int result[100];

    // Call the provided lambda function 99 times with blocks of size 1
    nt::parallel_for(
        nt::blocked_range<uint32_t>(/* begin = */ 0, /* end = */ 100, /* block_size = */ 1),

        // The callback is allowed to be a stateful lambda function
        [&](nt::blocked_range<uint32_t> range) {
            for (uint32_t i = range.begin(); i != range.end(); ++i) {
                printf("Worker thread %u is starting to process work unit %u\n",
                       pool_thread_id(), i);

                // Write to variables defined in the caller's frame
                result[i] = i;
            }
        }
    );
}
```

Small amounts of work that only consist of a single block will immediately be
executed on the calling thread instead of involving the thread pool. Exceptions
occurring during parallel execution will be captured and re-thrown by
``nt::parallel_for``.

### Parallel for loops (asynchronous)

Parallel `for` loops can also run asynchronously—in that case, the function
immediately returns a ``Task *`` handle that can be used to wait for
completion, or to schedule *child tasks*, whose execution will be delayed until
all parents have completed.

```cpp
template <typename T, typename Func>
Task *parallel_for_async(const blocked_range<T> &range, Func &&func,
                         std::initializer_list<Task *> parents = { },
                         Pool *pool = nullptr);
```

The returned task handle must eventually be released using the functions
``task_release(Task *)`` (which is instantaneous) or
``task_wait_and_release(Task *)`` (which blocks until the task has terminated).
A failure to do so will leak memory.

Example:
```cpp
#include <nanothread/nanothread.h>

namespace nt = nanothread;

int main(int, char **) {
    // Schedule task 1
    Task *task_1 = nt::parallel_for_async(
        nt::blocked_range<uint32_t>(/* ... */),
        [&](nt::blocked_range<uint32_t> range) { /* ... */ }
    );

    // Schedule task 2
    Task *task_2 = nt::parallel_for_async(
        nt::blocked_range<uint32_t>(/* ... */),
        [&](nt::blocked_range<uint32_t> range) { /* ... */ }
    );

    // Schedule task 3 ...
    Task *task_3 = nt::parallel_for_async(
        nt::blocked_range<uint32_t>(/* ... */),
        [&](nt::blocked_range<uint32_t> range) { /* ... */ },
        { task_1, task_2 } // ... <- but don't execute until these tasks are done
    );

    task_release(task_1);
    task_release(task_2);
    task_wait_and_release(task_3);
}
```

If a task only consists of single-threaded work that cannot easily be converted
into a parallel ``for`` loop, the function ``do_async`` provides an more
convenient interface that is analogous to ``parallel_for_async`` with a
``blocked_range`` of size 1.

```cpp
template <typename Func>
Task *do_async(Func &&func, std::initializer_list<Task *> parents = {},
               Pool *pool = nullptr);
```

## Examples (C99 interface)

The following code fragment submits a single task consisting of 100 work units
and waits for its completion.

```c
#include <nanothread/nanothread.h>
#include <stdio.h>
#include <unistd.h>

// Task callback function. Will be called with index = 0..99
void my_task(uint32_t index, void *payload) {
    printf("Worker thread %u is starting to process work unit %u\n",
           pool_thread_id(), index);

    // Use payload to communicate some data to the caller
    ((uint32_t *) payload)[index] = index;
}

int main(int argc, char** argv) {
    uint32_t temp[100];

    // Create a worker per CPU thread
    Pool *pool = pool_create(NANOTHREAD_AUTO);

    // Synchronous interface: submit a task and wait for it to complete
    task_submit_and_wait(
        pool,
        100,     // How many work units does this task contain?
        my_task, // Function to be executed
        temp     // Optional payload, will be passed to function
    );

    // .. contents of 'temp' are now ready ..

    // Clean up used resources
    pool_destroy(pool);
}
```

Tasks can also be executed *asynchronously*, in which case extra steps must be
added to wait for tasks, and to release task handles.

```c
/// Heap-allocate scratch space for inter-task communication
uint32_t *payload = malloc(100 * sizeof(uint32_t));

/// Submit a task and return immediately
Task *task_1 = task_submit(
    pool,
    100,       // How many work units does this task contain?
    my_task_1, // Function to be executed
    payload,   // Optional payload, will be passed to function
    0,         // Size of the payload (only relevant if it should be copied)
    nullptr,   // Payload deletion callback
    0          // Enforce asynchronous execution even if task is small?
);

/// Submit a task that is dependent on other tasks (specifically task_1)
Task *task_2 = task_submit_dep(
    pool,
    &task_1,   // Pointer to a list of parent tasks
    1,         // Number of parent tasks
    100,       // How many work units does this task contain?
    my_task_2, // Function to be executed
    payload,   // Optional payload, will be passed to function
    0,         // Size of the payload (only relevant if it should be copied)
    free,      // Call free(payload) once this task completes
    0          // Enforce asynchronous execution even if task is small?
);

/* Now that the parent-child relationship is specified,
   the handle of task 1 can be released */
task_release(task_1);

// Wait for the completion of task 2 and also release its handle
task_wait_and_release(task_2);
```

## Documentation

The complete API is documented in the file
[nanothread/nanothread.h](https://github.com/mitsuba-renderer/nanothread/blob/master/include/nanothread/nanothread.h).

## Technical details

This library follows a lock-free design: tasks that are ready for execution are
stored in a [Michael-Scott
queue](https://www.cs.rochester.edu/u/scott/papers/1996_PODC_queues.pdf) that
is continuously polled by workers, and task submission/removal relies on atomic
compare-and-swap (CAS) operations. Workers that idle for more than roughly 50
milliseconds are put to sleep until more work becomes available.

The lock-free design is important: the central data structures of a task
submission system are heavily contended, and traditional abstractions (e.g.
``std::mutex``) will immediately put contending threads to sleep to defer lock
resolution to the OS kernel. The associated context switches produce an
extremely large overhead that can make a parallel program orders of magnitude
slower than a single-threaded version.

The implementation catches exception that occur while executing parallel work
and re-throws them the caller's thread (this part is of no relevance for
software written in C99).

The functions ``task_wait()`` and ``task_wait_and_release()`` do not just
wait---they spend the wait time fetching and executing work from the task
queue, which has two implications: first, it is not wasteful to wait for the
completion of another task while executing a task. Second, the thread pool can
be set to a size of zero via ``pool_create(0)`` or ``pool_set_size(pool, 0)``,
in which case the program will still run correctly without launching any
additional threads.
