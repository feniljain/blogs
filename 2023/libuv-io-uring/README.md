# [Draft] Exploring libuv's io_uring integration

Recently, an interesting PR got merged in libuv: https://github.com/libuv/libuv/pull/3952, it's about adding io_uring support. This PR sparked relatively wider attention cause io_uring is known for it's performant unified APIs around file and networking operations. This support could provide some really good performance benefits over different projects including v8, chrome's javascript engine powering chrome and nodejs.

This PR only adds support for a subset of file operations but is super interesting neverthesless. Original effort started in 2019 with this PR: https://github.com/libuv/libuv/pull/2322. It saw some good progress over the years but sadly died down in 2021.

Currently, libuv uses a thread pool for file operations. It gives all the file descriptors(fd) to epoll and waits in the event loop to get any ready events, as soon some fd is ready for performing IO, event loop is notified and task is given to a thread from thread pool. This is how readiness based APIs work, they inform you once an operation is ready to be done, and you then perform the task. In this case, it can be any file operation like reading, writing, etc. [ needs proof ]

io_uring works in a slightly different manner, one keeps giving io_uring tasks, here tasks mean reading file, writing file, etc. itself. io_uring informs back when the tasks are complete. So you can imagine io_uring's work as readiness API work + performing the task too and then notifying when it's done. These APIs are known as completion based APIs. [ This is incorrect cause io_uring provides both readiness and completions based API, source: https://github.com/tokio-rs/mio/issues/1591  ]

One can see what is the difference between readiness and completions based APIs. As mentioned before libuv is built around epoll, which is a readiness based API, how did it integrate a completion based API in it's architecture. This was the question which led me to take a look at PR in detail and understand how it was implemented.

## Some understanding about io_uring

io_uring internally maintains two ring buffers in shared memory, one is a submission queue and one is a completion queue. User submits tasks in the submission queue and get's the completed tasks/results, etc. on the completions queue. This provides an nice async interface around both file and networking operations. In fact, io_uring is more of an async interface for different IO operations, file operations is just a part of it. Because of the way it is designed from ground up and addressing the very question of what it means to provide an async framework it is able to provide a general interface for variety of different IO operations. [ needs proof ]

libuv currently uses two io_uring syscalls:

- io_uring_setup
- io_uring_enter

Let's understand each one of them more:

- `io_uring_setup` is the init call for io_uring, this call sets up submission and completion queues (sq and cq). It's params inlcudes least number of entries to be preallocated in queue and other params for conveying informating about ring buffers.

Our story starts here: https://github.com/bnoordhuis/libuv/blob/26c79a942b92573a1388c0ee8a6ad4397f009318/src/unix/linux.c#L378, this is the function which calls `io_uring_setup` here: https://github.com/bnoordhuis/libuv/blob/26c79a942b92573a1388c0ee8a6ad4397f009318/src/unix/linux.c#L406-L416 . Checking the params passed by libuv, they pass 64 in least entries field and they pass `UV__IORING_SETUP_SQPOLL` in params. The param passed is interesting, cause it allows us to understand how io-uring works under the hood.

io_uring handles task in two different ways:

- Polling:
- Internal polling:

- Interrupts: This is the default mode. As the name suggests, this sets io_uring for interrupt driven IO.
- Polling: In this mode io_uring polls for IO readiness, and then performs when it is ready. Exactly what we discussed above.
    - In this mode user would have to poll the cq using `io_uring_enter` for getting completions.
    - This mode is useful when latency is of importance, more than CPU cycles.
- Kernel polled: In this mode.

<!-- =========================================== -->


## References:

- https://www.sobyte.net/post/2022-04/io-uring/
- https://www.scaler.com/topics/nodejs/libuv/

## Introduction to libuv and io_uring and how they are related?

libuv provides a cross-platform library for high performance async IO. NodeJS and many other prominent projects use libuv under the hood for everything from event loop, networking to thread pool,  high resolution clock, etc.

io_uring

I am intentionally keeping the intro short assuming reader knows enough about both of them,

Currently on linux it uses thread pools for file


