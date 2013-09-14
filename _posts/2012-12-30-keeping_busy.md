---
layout: default
title: Keeping Busy
---

Over the holidays I decided it might be fun to have another poke at my
game/engine project that is in a permanent state of unfinishedness and flux.
Of course I haven't managed to solve that fundamental problem but I did manage
to get a reasonable first pass down on a task parallelism system. So I guess
we can mark that down as a success!

The whole thing is pretty basic and the design almost entirely ripped off from
Niklas Frykholm's [blog post][bitsquid-task-management] on Bitsquid's system. I
also read a lot of Charles Bloom's [thoughts][cbloom-good-threading] on the
matter, and promptly ignored most of his good advice with the vague idea that
I'll fix up the remaining issues eventually.

**Design**

So anyway the task system revolves around a global 'staging area' of tasks
combined with a priority queue of task IDs (32 bit handles also
[ripped off][bitsquid-id-lookup] from Bitsquid) that workers draw from. Each
task consists of a work function and a user data pointer. This leaves the
interesting bits of multithreading up to the programmer.

The entire dependency chain is built on-the-fly as in Bitsquid's system. In
order to allow interesting dependency graphs each task can have a parent task
and/or a task to schedule when the task completes.

When submitting a task with a parent we increment the parent's `open_children`
counter and whenever a task with a parent completes we decrement the parent's
counter. Once the counter reaches zero the task is complete. Note that the
task's own work item is considered a child in order to keep the code simple.

Whenever a task finishes (i.e the `open_children` counter reaches zero) we
check whether the task has an `allows_to_start` field in which case we enqueue
the linked task. This kind of backwards dependency keeps the code nice and
and simple at the unfortunate cost of making code that uses the task system
read backwards.

**API**

You submit a number of tasks, optionally giving a parent, and get a number of
handles back. This stages the tasks but does not queue them for execution.

```c
    typedef struct RdxTask
    {
        void *user_data;
        void (*work_fn)(struct RdxTaskPool *pool, void *user_data);
        RdxTaskHandle allows_to_start;
    } RdxTask;

    void
    rdx_task_submit(RdxTaskPool *self, RdxTaskHandle parent, RdxTask *tasks,
            RdxTaskHandle *out_task_handles, size_t num_tasks);
```

Note that we pass the parent task only once per submit call, but the
`allows_to_start` is defined once per task. This is just simpler to code, it
means I can lookup a single task in submit and add `num_tasks` to the
`open_children` counter. It also covers the main use-case where you'd be using
a single parent to control a number of children. `allows_to_start` does not
make much sense as applied to a group of tasks however, so we pass that as part
of the task. We also pass the task pool to the worker function so workers can
themselves stage, enqueue and even wait on more tasks.

Once we've submitted enough tasks to get running we can enqueue tasks which
schedules them for execution in the priority queue.

```c
    void
    rdx_task_enqueue(RdxTaskPool *self, int priority, RdxTaskHandle task);
```

We give the priority here once again to simplify the implementation. Each task
scheduled from another task inherits that task's priority. It's yet to be seen
whether this is a reasonable approach or even if a priority is worth having at
all.

Now we have some tasks and they're probably off getting executed so we might
want to block this thread until they do complete.

```c
    void
    rdx_task_wait(RdxTaskPool *self, RdxTaskHandle task);
```

Wait just spins checking if the task given still exists and if so, if there are
work items to dequeue and execute. This busy loop has a nasty case if a thread
is waiting on the last enqueued task and another thread is executing but it
works well enough for now.

**Issues**

At the moment the entire staging area and handle pool are protected by a single
mutex which lends to quite a bit of contention (and hence context-switching).
The reason for the single lock is that we keep the staging area array tightly
packed. The packing means that any task finishing could lead to any other task
having its position in the array changed and hence its handle being updated.
This is probably just a bad design choice. I kept the idea of the packed array
from an earlier implementation where I needed to iterate over the staging area
but without that requirement the only benefit is a very simple submit
procedure. I could simplify the handle system and improve the locking
situtation by flattening all that code and using a sparse array and freelist.

The task queue is also a potential performance bottleneck although since I'm
only storing small bits of data (64 bits per entry) and it's locked separately
it's not a huge issue yet. Potentially worthwhile just ditching the priority
rubbish and simplifying the whole thing.

I'm also only using a thin wrapper over the OS' semaphores, this shouldn't be
an issue on linux but as I understand it semaphores in Windows are pretty
heavy-weight items. It may eventually be worth wrapping those up benaphore
style as [mentioned by][cbloom-fast-semaphore] Charles Bloom.

**Finally**

The other thing that's completely ignored at the moment is thread affinity. I'd
like to design the rest of the engine so that hopefully the need for such is
low however there's always the frustrating bits of OpenGL where you have to
render on the same thread which you create the window. For the moment I plan to
serialise the final render and just run the important bits on the main thread
outside the task system. Eventually though thread affinity support would be
nice.

Well there's my post-implementation brain dump, maybe it makes some sense. At
this stage I'm likely going to keep the system as-is and move onto other things
since it seems to work OK and it's not really worth doing anything else until I
can collect some science on it performs in practice.

[bitsquid-task-management]: http://bitsquid.blogspot.com.au/2010/03/task-management-practical-example.html
[cbloom-good-threading]: http://cbloomrants.blogspot.com.au/2011/07/07-13-11-good-threading-design-for.html
[bitsquid-id-lookup]: http://bitsquid.blogspot.com.au/2011/09/managing-decoupling-part-4-id-lookup.html
[cbloom-fast-semaphore]: http://cbloomrants.blogspot.com.au/2011/12/12-08-11-some-semaphores.html


