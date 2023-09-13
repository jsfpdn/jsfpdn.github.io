---
title: "Parallel Programing With OpenMP"
date: 2020-10-25T14:05:10+02:00
tags: ["c++", "parallel programming", "programming"]
summary: "Messing around with the OpenMP library."
---

OpenMP is an API specification for parallel programming
using a **single program, multiple data** paradigm.
It is available for C/C++ and Fortran programs.
It provides programmers a set of high level parallel-programming techniques
via compiler directives and runtime functions.

"I am programming in C++ which has threads already in the standard library or
I can just leverage the `pthread` library, how can OpenMP help me?"
This was the exact question I asked myself when I first dabbled with the OpenMP library.
The fact is, OpenMP does not just introduce the concept of threads,
it provides pretty powerful techniques and optimizations on top of threads,
that are ready for programmers to use.
OpenMP offers **cyclic iteration distribution among threads**
(assigning iterations in a round-robin fashion to threads),
**various synchronization techniques**,
such as barriers and data synchronizations, **work-sharing constructs** and much more.

## OpenMP Basics

* When compiling code with OpenMP, add `-fopenmp` argument when compiling
  (or `-libopenmp` on MacOS)
* Directives look like `#pragma omp directive-name [clause, ...] newline` followed
  by a structured block of code to which the directive applies to
* Set number of threads with `omp_set_num_threads`
  (Note that this may differ from the actual number of threads that will be assigned)
  or with the environment variable `OMP_NUM_THREADS`
* Get number of threads with `omp_get_num_threads`.
  Be sure to call this code only in the parallel regions.
  If this function is called outside of the parallel region,
  than `1` is returned, since there's only a master thread running.
* Get id of current thread with `omp_get_thread_num`.

## Parallelizing a for loop

This is the bread and butter of parallel programming.
You have a for loop and you want to make it run faster using threads.
The basic idea is that iterations will be split among the threads.
In order to be able to do this, some conditions must be met beforhand:
(1) **iterations must be independent**, (2) **`goto` must not be used in the parallel region**,
(3) **loop variable must be an integer**.
There are other conditions, but they don't concern us at the moment.
Let's just look how it can be implemented in the most basic way
and we'll work towards better OpenMP approach.

```cpp
#include <omp.h>

void foo();

void main() {
    int UNTIL = 1000;

    #pragma omp parallel
    {
        int thread_id = omp_get_thread_num();
        int threads = omp_get_num_threads();
        for( int i = thread_id; i < UNTIL; i += threads ) {
            foo(); // do some computation
        }
    }
}
```

The code above defines a constant `UNTIL`, which is the number of iterations.
Next, the `#pragma omp parallel` **spawns a team of threads** and each thread will enter the structured block.
Note that we did not specify the number of threads we wish to spawn.
It is possible that no extra thread will be spawned and we will enter this region
with just the master thread we already had.

The for loop runs in parallel since every thread has some iterations dedicated to itself.
So for a thread with id = 3 with 8 threads running in parallel,
this thread will work with iterations `i = 3, 11, 19, 27, 35...` with the last iteration being `i = 993`.
This will result in nice and even distribution of the workload
if the function `foo()` runs for the same amount of time for any iteration.

Let's refactor the code a bit. This is pretty standard use case,
so of course OpenMP has some constructs making this a bit more comfortable.

```cpp
#include <omp.h>

void foo();

void main() {
    int UNTIL = 1000;

    #pragma omp parallel for
    for( int i = 0; i < UNTIL; i++ ) {
        foo(); // do some computation
    }
}
```

Now this is much better. All we had to do is add directive before the for loop.
Notice that the variable `i` used for iterating is the same as for sequentiall code,
not like when we had to mess around with the thread id's and total number of threads before.

Writing for loops like this has some other benefit -
we can manually adjust how we want to schedule the workload among the threads.
This can be done with a special `schedule` clause that can be added right after the `for` in the directive.
More about that can be found [here](https://computing.llnl.gov/tutorials/openMP/#DO).

### Data Sharing

It is possible to share data among threads. One of the basic constructs in OpenMP
are the `shared`, `private`, `firstprivate`, and `lastprivate` clauses that can be added to directives.

**`shared`** is the default visibility of global data among other threads.
In our previous case, the variable `UNTIL` was shared.
It is visible and even modifiable by every thread in the team.

**`private`** can be used to tell OpenMP that some global variable should be copied into each thread
and made local to that thread.
Note that OpenMP just replaces the references from the global variable to the local copy of the variable.
Should be assumed to be uninitialized for each thread.

**`firstprivate`** works the same way as `private` does, but initializes the variable automatically.

**`lastprivate`** combines the behavior of `private` clause with copy
from the last loop of the iteration to the original variable.

```cpp
#include <omp.h>

void foo(double);

void main() {
    int UNTIL = 1000;
    int i;
    double pi = 3.1459;

    #pragma omp parallel for shared(UNTIL) private(i) firstprivate(pi)
    for( i = 0, j = 100; i < UNTIL; i++ ) {
        foo(pi); // do some computation
    }
}
```

Here we declared that the variable `UNTIL` is shared,
`i` is local to every thread in the team (note that we defined it right after the entry to `main`,
but the initialization is done in the `for` clause), and `pi` is local to every thread,
but it's value is well defined and initialized to 3.1459 in every thread.

We can also change the default visibility with the `default(none|shared)`.
`none` requires that the programmer explicitly scopes all variables. Very useful when debugging.

## Synchronization

OpenMP provides various synchronization techniques for use that are ready to use.
Some of the main directives are `CRITICAL`, `BARRIER`, `ATOMIC`, and `TASKWAIT`.

### Critical Section

The `#pragma omp critical [name]`  directive specifies a region of code that can be executed by only
one thread at a time.
This means that if some thread is already within critical block,
every other thread will block attempting to entry the critical block
until the first thread exits the critical region.

The way critical regions work is that **there's one lock for every named critical block**.
Important thing to note is if we omitt the name of the critical block,
it becomes an unnamed critical block and it will share lock with every other critical block
that's in the code. This means that **2 seemingly unrelated unnamed critical blocks can cause some
serious performance degradation because of this**.

```cpp
#include <string>

double foo(double);
void write_to_file(std::string, double);

void main() {
    double pi = 3.1459;
    std::string output = "./out_file.txt";

#pragma omp parallel shared(pi, output)
    {
        double result = foo(pi); // do some computation
        // make sure that the result will be correctly written to the file
#pragma omp critical
        write_to_file(output, result);
    }
}
```

### Barrier

Barrier directive synchronizes all threads in the team.
Every thread that reaches the barrier will block,
wait for rest of the team to reach the barrier,
and only when every thread is at the barrier, the execution will continue.

```cpp

double foo(double);
void write_to_file(std::string, double);

void main() {
    double pi = 3.1459;
    std::string output = "./out_file.txt";
#pragma omp parallel shared(pi)
    {
        double result = foo(pi);
        // wait for all the computations to finish
#pragma omp barrier
        write_to_file(output, result);
    }
}
```

*Note: There's an implicit barrier at the end of a parallel and sections regions.
Only the master thread will continue the execution past this point.*

## Work Sharing

Besides the iterative work sharing construct described above,
there's another similar directive; `SECTIONS`.
Sections help to split different work load among threads.
It specifies that the enclosed sections of the code should be distributed
among the threads in the team.
If there are more threads than sections,
than some the threads will not partake in the parallel execution.

```cpp
#include <iostream>

double foo(double);
double bar(double);
double baz(double);

void main() {
    double pi = 3.1459;
    double fst, snd, thd;
#pragma omp sections
    {
        std::cout << "Entering sections..." << std::endl;
#pragma omp section
        fst = foo(pi);
#pragma omp section
        snd = bar(pi);
#pragma omp section
        thd = baz(pi);
    }
    std::cout << "Results: " << fst << ", " << snd << ", " << thd << std::endl;
}
```

## Sources

* [OpenMP tutorial](https://computing.llnl.gov/tutorials/openMP)
* [Exquisite video series by Tim Mattson](https://www.youtube.com/playlist?list=PLLX-Q6B8xqZ8n8bwjGdzBJ25X2utwnoEG),
  Senior Principal Engineer @ Intel developing OpenMP
