---
layout: post
title: Implementation of a fiber based job system as stb like single header library
---
I spend some time implementing my own fiber based job system. It is very similar to the job system Christian Gyrling [presented](https://www.gdcvault.com/play/1022186/Parallelizing-the-Naughty-Dog-Engine) and also [this blog post](https://ourmachinery.com/post/fiber-based-job-system/) from our machinery helped in creating this library. This blog post mostly covers only the important parts in how I implemented this library.

First of these were my requirements:
- You should be able to yield jobs at any point of its execution
- Tasks can create and wait on subtasks
- No allocations after initialization of the jobsystems context
- It should work on Windows, MacOS and Linux
- The jobsystem should be implemeted as a STB style single header file (kit_job.h)
- Run optional without fibers via defines

First we will look now into the implementation of the job system and after that a bit into the more interesting platform specific parts.

## Implementation
All memory needed is allocated on initialization of the jobsystem and in as little batches as possible, even the thread local data is initialized as an array. For Threads, Locks, TLS and atomics system primitives where used. All thread worker share one job queue which is accessed currently by aquiring a lock.

### Queues & Pools
All pools and queues are shared among main thread and worker thread.
On startup the free-queues are filled up with all indexes from their corresponding pool, so in the example, to allocate or free a counter you need only to touch the freeCounterInidices queue and then return the pointer to counter in the counter pool.
Here are all queues & pools used in kit_job.h
```C
ktj_a32 *counters; // pool
ktj_a32 *freeCounterInidices; // queue
ktj_Job *jobs; // queue
uint32_t* freeCounterIndicies; // queue
ktj_Fiber *fibers; // pool
uint32_t* freeFiberIndicies; // queue
sleeping_fiber *sleepingFibers; // queue
```

Queues are defined as ring buffers, or to be more precise they are ring arrays.
Each queue has an variable that points to the index where to put in the next item (in), one that contains the index which item to pull the next item (out) and the size of the queue (count).
```C
ktj_Job *jobs;
int jobsCount;
int jobsIn;
int jobsOut;
```
The indices can be bigger than the queue size. To access an item in the queue the modulo operator is used. This is basically the ring buffer property of the queue.

So in case you want to push an element to the job queue you store it at the index of the queues "in" variable and increase it by one.

```C
/* ... */
int myItemIndex = ctx->jobsIn;
ctx->jobs[myItemIndex % ctx->jobsCount] = (ktj_Job) {};
ctx->jobsIn += 1;
/* ... */
```
I will not go into details how to make this lock free, there is an excelent [post](https://blog.molecular-matters.com/2015/09/25/job-system-2-0-lock-free-work-stealing-part-3-going-lock-free/) on the molecular-matters blog about this

Also, you may see a problem here, the jobsIn and jobsOut out could overflow at some point. If that ever gets to be a problem we can always use an int64. It will probably take years of non-stop running before these overflows.

### Job from its creation to its destruction
When creating a joblist that should run, a counter is pulled from the freeCounterIndicies queue and it is set to the numbers of jobs in the joblist. After that all jobs gets pushed to the jobs queue the index of the counter gets stored as well. Other job runner theads can now run the jobs that just has been created. A id will be returned the represents the counter, finally waitForCounter can be called, this will yield the fiber and put it into the queue for fibers that wait for other jobs. The only exception is when the counter already reached zero. In this case the counter gets pushed to the freeCounterInidices queue and the fiber continues running.

When a job is done it decreases the counter by one. If the counter reaches zero it tries to run the task that awaits for the jobs to finish. In that case the counter gets pushed to the freeCounterInidices queue.

### Fibers Windows
Windows has direct [support for fibers](https://docs.microsoft.com/en-us/windows/desktop/procthread/using-fibers) so its pretty much a nobrainer what to use here. Implementation was also pretty easy, they have a similar api to threads.

### Fibers everything else (except WASM)
On Linux and MacOS we have makecontext and swapcontext to implement fibers but these are not part of Posix and in Xcode get some deprecation warnings when you try to use them.
Using deprecated functions does not look like the best idea to me, but you can implement fiber switching in x86 assembly directly.
In x86 you got the assembly command jmp which is also supported by other assembly languages. With jmp you can do what it says you can jump to a specific execution pointer.
Before you jump to another fiber you save the current one, so you push the current registers to the stack.
To make things easier I use some assembly from the boost library which precisely does this.
The question is now, how does that fit into a single header library?
- MSVC does not support inline x86 assembly... but that's not problem because windows already has a fiber solution
- Using inline assembly... can be problematic just using it inside a fuction can be problematic but Clang/GCC has a solution for this:
```C
__attribute__((naked)) void jump(void *jumpTarget) {
    /* ... */
}
```
Also boost provides assembly for arm32, arm64 so with that all current important architectures are covered.

### Running without fiber
On some platforms it may be impossible to yield, for example in Web Assembly. For these platform I would like to be at least able to use the job system.
To enable this, a running task that awaits that waits for other jobs will run other task with the same heap.
This has a big downside because now, even through its wait counter went to zero, it can't continue till the jobs are finished that run atop of it. This makes the job system obviously behave quite differently compared to using fibers.

## Usage
There is only a single file you need to add to your project kit_job.h. Somewhere inside an implementation file you add this block.
```C
#define KIT_IMPL
#include "kit_job.h"
```
Then you call its initialization call and you are ready to go.

```C
int main() {
	/* ... */
	ktj_Context* ctx = ktj_createContext(&(ktj_ContextDesc) { .maxJobs = 256 });
	/* ... */
}
```
All memory that the job system needs to run is defined at this point, you can also pass a custom allocator.

Finally, you can create jobs and yield, you can even yield on your main thread, its basically just another worker thread.

To create a job you provide a function as entry point and user data.

```C

KIT_JOB_ENTRYPOINT(myJobFn) {
	/* ... */
}

void runJob() {
	int results[3] = {0};

	ktj_JobDesc jobs[] = {
		{myJobFn, &results[0]},
		{myJobFn, &results[1]},
		{myJobFn, &results[2]},
	};

	ktj_counterId jobsHandle = ktj_runJobs(ctx, jobs, 3);

	ktj_waitForCounter(ctx, jhandle);
}
```
When calling ktj_waitForCounter the fiber gets yield and will continue when all jobs are finished. In the current implementation this function has to be called, otherwise counter will not be freed.

## Future work
The most obvious thing to do is to look at the remaining locks and try to exchange these with lockless implementations.
Per thread job queues with job stealing would be an interesting experiment as well, also I have some ideas on how to make the sleeping job queue obsolet.
First and formost I would like to actually use this libary in an project to see whats missing and where may be some common usecases which are not covered by the job system.
