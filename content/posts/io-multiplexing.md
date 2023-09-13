---
title: "I/O Multiplexing - Event-Based Notifications in UNIX"
date: 2019-12-12T20:11:08+01:00
tags: ["programming", "unix", "c"]
summary: "Quick look into kernel's notification systems"
---

Let's say we have a webserver that opens a socket for every new connection.
When we want to somehow manage these connections,
we want to know if the socket sent us some new data (e.g. is ready to be read from)
or is awaiting some data (is ready to be written to). How do we even do this?

Okay, I got it! Let's call `fork` and create a new thread for every socket!
This way, the threads do not mess with each other,
each thread has it's own stack and programming logic.
But what happens when there are 1000 active users?
Suppose each thread allocates 2MB, that's 2GB allocated in total. Yikes.

We could maybe optimize the previous approach by having some arbitrary number of threads
at all times and simply moving the connections between the threads,
but this would cause some unwanted programming logic.
The best approach would be to let the kernel handle all of this and just tell us
whenever some connection (read file descriptor) is ready to be read from/written to.

## `select`

`select` is just the right system call for this kind of operation.
This is the function signature we discover when we dive into the man page:

```c
int select(int nfds, fd_set *r, fd_set *w, fd_set *r, struct timeval *timeout);
```

This is what the parameters mean:

* `int nfds` - how many descriptors should be checked in each set (`readfds`, `writefds` & `errorfds`)
* `fd_set *r` - set of descriptors we are interested in reading from
* `fd_set *w` - set of descriptors we are interested in writing to
* `fd_set *r` - set of descriptors we are interested in checking some exceptions
* `struct timeval *timeout` - minimum interval to wait for the selection to complete

When we want to use this system call,
we have to exactly know what descriptors we want to read from,
write to or check if some exception happend.
Return value specifies if nothing has changed (returned 0)
or data in the specified descriptors is ready to be inspected (any positive integer).

How does the system call `select` actually work? When we call `select`,
it goes through ALL of the file descriptors we provided in the read, write and error `fd_set`s.
If there were no changes to the data, it tells the kernel to wake it up when the data changes
or when the `timeout` was reached and goes to sleep.
When awakened, it checks all of the descriptors again, if there were any changes to the data.

Here's a simple example waiting an input from `stdout` and just mirroring it back to the user:

```c
#include <stdio.h>
#include <string.h>
#include <sys/select.h>
#include <unistd.h>

int main() {
    int buffsize = 10;
    char buff[buffsize];
    int ret, sret, wret;
    int rfd = 0, wfd = 0;

    fd_set readfds, writefds;
    struct timeval timeout;

    for (int i = 0;; i++) {
        FD_ZERO(&writefds);
        FD_ZERO(&readfds);
        FD_SET(rfd, &readfds);
        FD_SET(wfd, &readfds);

        timeout.tv_sec = 5;
        timeout.tv_usec = 0;

        printf("ITERATION #%d\n", i);
        printf("calling select...\n");
        sret = select(10, &readfds, NULL, NULL, &timeout);
        printf("select has been called...\n");

        if (sret == 0)
            printf("did not receive anything\n\n");
        else {
            memset((void *)buff, 0, buffsize);
            ret = read(rfd, (void *)buff, buffsize);
            if (ret != -1) {
                wret = write(wfd, (void *)buff, buffsize);
                if (wret != -1) {
                    printf("\nmirrored data to the tty...\n\n");
                } else {
                    perror("write");
                }
            } else {
                perror("read");
            }
        }
    }
}
```

Here's a link to my [GitHub Gist](https://gist.github.com/josefpodany/948f9b941806682e46ba4920080eca9c).
I have added some `printf`s just so it's easier for us to follow the program when it's running.

I highly encourage you to run the program.
We will be innactive during the first iteration, doing nothing,
and writing `thisistextstuff` to the terminal during the second iteration:

```text
ITERATION #0
calling select...
select has been called...
did not receive anything

ITERATION #1
calling select...
thisistextstuff
select has been called...
thisistext
mirrored data to the tty...

ITERATION #2
calling select...
select has been called...
stuff

mirrored data to the tty...

ITERATION #3
calling select...
^C
```

The `ITERATION #0` took exactly 5 seconds, as expected from `timeout.tv_sec`,
because the `select` was not awakened by some change in the content of the file descriptor.
In the `ITERATION #1`, we wrote `thisistextstuff` to the terminal
and we immediately saw the input mirrored in the terminal.
This is because the system call `select` was awakened by the kernel.
Because the buffer was small during the reading from the file (10 bytes exactly),
we read only `thisistext`, the pointer was just moved ahead.
`ITERATION #2` started and the `select` call noticed that there are still some data that we have not processed yet, hence reading rest of the data from the buffer and mirroring `stuff` to the terminal.

Keep in mind that `select` is desctructive to the structs of file descriptors passed in -
we have to refill the sets using `FD_ZERO` and `FD_SET`.

## `poll`

`poll` comes in to "fix" some of the issues of `select`. This is the function signature:

```c
int poll(struct pollfd fds[], nfds_t nfds, int timeout);
```

Similiar to `fd_set`, `pollfd` struct is a set specifying all the file descriptors we are interested in.
`nfds_t` is the size of the intersted set.

Here, the `poll` is not desctructive to the `pollfd` parameter,
meaning we don't have to refill the sets before every call.
This spares us some programming logic on our side,
but creates some unnecessary overhead, because the kernel has to allocate and copy the structs, resolving in a bit worse performance than `select`.

But even this implementation does not yield the optimal performance.
This is because when the kernel wakes `select` or `poll` up,
it does not provide the information about WHICH file descriptor's contents
have changed, rather than say SOME of them have changed.
`select` or `poll` then have go through all the file descriptors provided in
`fd_set` or `pollfd` and
decide for themself in a linear manner in regards to an input.
Now we know that the problem is not in the implementation,
but in the design.

The behavior of `select` and `poll` can be described by a pattern
called **state-based view** -
the kernell informs the application of the current state of a file descriptor.

## Event-Based View

Quoting [a paper by Banga et al.](https://static.usenix.org/event/usenix99/full_papers/banga/banga.pdf),
"*An event-based view, in which the kernel informs
the application of the occurrence of a meaningful
event for a file descriptor (e.g., whether new data
has been added to a socket's input buffer).*"

This is excatly what we needed!
Instead of the kernel telling us that something's changed,
it tells us directly WHICH file descriptor's contents have changed,
sparing us the biggest overhead we've encountered.
This is where `kqueue` and `epoll` come in play.

## `kqueue` (FreeBSD)

This is the `kqueue` function signature:

```c
int kqueue(void);
```

Oh...kay? Let's have a look at `kevent` function signature described in the same man page:

```c
int kevent(
    int kq,
    const struct kevent *changelist,
    int nchanges,
    struct kevent *eventlist,
    int nevents,
    const struct timespec *timeout
);
```

The `kqueue` system call only allocates a kqueue file descriptor -
it is a generic method of notifying the user when a kernel event happens.
`kevent` is the main takeaway here; it specifies the interesting conditions to be notified about.

`EV_SET` macro is also important - it helps us initializing a kevent structure (there are also `EV_SET64` and `EV_SET_QOS` alternatives).
This is what it looks like:

```c
EV_SET(&kev, ident, filter, flags, fflags, data, udata);
```

* `&kev` - `kevent` structure.
* `ident` - Value used to identify the source of the event (usually a file descriptor).
* `filter` - Identifies the kernel filter used to process this event.
* `flags` - Actions to perform on the event.
* `fflags` - Specify, what events we are interested in.
* `data` - Filter-specific data.
* `udata` - Opaque user data identifier.

We can specify what events we're interested in simply by bitwise ORing the following masks:

* `NOTE_DELETE` - `unlink()` has been called on the file descriptor.
* `NOTE_WRITE` - Write occured to the file referenced by the file descriptor.
* `NOTE_EXTEND` - The file was extended.
* `NOTE_ATTRIB` - The attributes of the file changed.
* `NOTE_LINK` - The link count on the file changed.
* `NOTE_RENAME` - The file has been renamed.
* `NOTE_REVOKE` - Access to the file was revoked via `revoke(2)`.
* `NOTE_FUNLOCK` - The file was unlocked by calling `flock(2)` or `close(2)`.

Let's implement a function which notifies us when a certain file has changed.
Here's what the most basic implementation could look like:

```c
#include <sys/event.h>
#include <sys/time.h>

#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    int f, kq, kev;
    struct kevent change;
    struct kevent event;
    struct timespec timeout;

    timeout.tv_sec = 5;
    timeout.tv_nsec = 0;

    kq = kqueue();
    if (kq == -1) perror("kqueue");

    f = open("./our_file", O_RDONLY);
    if (f == -1) perror("file");

    EV_SET(&change, f, EVFILT_VNODE, EV_ADD | EV_ENABLE | EV_ONESHOT, NOTE_DELETE | NOTE_EXTEND | NOTE_WRITE, 0, 0);

    for (int i = 0;; i++) {
        printf("ITERATION #%d\n", i);
        printf("calling kevent...\n");
        kev = kevent(kq, &change, 1, &event, 1, &timeout);
        printf("kevent has been called...\n");
        if (kev == -1)
            perror("kevent");
        else if (kev > 0) {
            if (event.fflags & NOTE_DELETE) {
                printf("File deleted\n");
                break;
            }
            if (event.fflags & NOTE_EXTEND || event.fflags & NOTE_WRITE) printf("File modified\n");
        }
    }

    close(kq);
    close(f);
    return EXIT_SUCCESS;
}
```

Before compiling and executing this source code, let's create a file called `our_file` right next to it.
When we run this program, it checks whether some changes (`NOTE_DELETED`, `NOTE_EXTEND` or `NOTE_WRITE`) occured.
If we try to add some data to the file in another terminal,
we should see that the program printed out the line `File modified`.
When we delete the file, the program prints out `File deleted` and exits.

## `epoll` (Linux)

`No manual entry for epoll` 😢

`epoll` is an alternative to `kqueue` on Linux systems.
It is described really well [here](https://medium.com/@copyconstruct/the-method-to-epolls-madness-d9d2d6378642).

## `epoll` vs `kqueue`

When it comes to comparing `epoll` and `kqueue`, there is one major difference:
epoll does not support not file-like 'objects' - processes, signals, timers, network devices.
`kqueue` can even receive a notification when a child process exits.

## Sources

* [Scalable Event Multiplexing: epoll vs. kqueue](https://people.eecs.berkeley.edu/~sangjin/2012/12/21/epoll-vs-kqueue.html)
* [The C10k Problem](http://www.kegel.com/c10k.html)
* [A Scalable and Explicit Event Delivery Mechanism for UNIX](https://static.usenix.org/event/usenix99/full_papers/banga/banga.pdf)
* [Linux Kernel Archive](http://lkml.iu.edu/hypermail/linux/kernel/0010.3/0003.html)
* [MegaPipe: A New Programming Interface for Scalable Network I/O](https://people.eecs.berkeley.edu/~sangjin/static/pub/osdi2012_megapipe.pdf)
