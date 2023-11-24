# msync

```msync``` is a wine patch set utilizing [Mach](https://web.mit.edu/darwin/src/modules/xnu/osfmk/man/) semaphores and (if available) the ulock kernel interface to provide efficient NT-synchronization primitive emulation for Wine on macOS.

It draws inspiration from ```esync``` and ```fsync```, particularly relying on shared memory code from ```fsync```.

## Features
* **Efficient Wait Operation:** Uncontended waits happen completely in user-space, with a dedicated Mach message pump handling all waiting synchronization tasks.
* **No File Descriptor Limitations**: Unlike ```esync```, ```msync``` does not have file descriptor limitations.
* **Dynamic Semaphore Pool**: Semaphores are sourced from a pool that dynamically adjusts based on application needs. With a high semaphore per process limit of 267597, it is unlikely for processes to be terminated by the kernel due to exceeding this number. Even in such cases, only the faulty process is affected, ensuring ```wineserver``` or other Wine processes remain uninterrupted.


## How to Use

**Enable msync:**
```
WINEMSYNC=1
```

**Adjust Queue Size (optional):** The default queue size of the server's receive mach port is 50
```
WINEMSYNC_QLIMIT=50
```

**Debugging:** Debug using flags similar to ```esync```. For example:
```
WINEDEBUG="+msync"
```

## How it works

The problem with accelerating NT synchronization primitives is that it needs to be possible to wait on multiple of these, and they need to work cross-process.

Originally, I intended to use `kqueue()`, since it supports multiplexing and seemed like a perfect fit, except that these kernel event queues are not inherited by a child created with fork, making them pretty useless for this...

Incidentally, this is also the reason why solutions like [epoll-shim](https://github.com/jiixyj/epoll-shim) do not work for providing an efficient (cross-process) emulation of eventfds on non-Linux systems. Still, [libinotify-kqueue](https://github.com/libinotify-kqueue/libinotify-kqueue) works perfectly, for example.

As an alternative, it is possible to do multiplexing with Mach port sets, but since every port can only have one receive right, this would mean that every process needs to create its own Mach port per object, and every other process (including wineserver) needs to be aware of all of these before they can signal an object.

This is indirectly exactly what the current `esync` implementation on macOS does with its pipe file descriptors to work around the issue. A `write()` on an fd gets translated by the kernel to a Mach message send, and a `read()` to a Mach message receive (and Mach port management is handled under the hood).

For msync, a slightly different approach was chosen, namely to mimic futex semantics with a separate thread managing wait (un-)registration and signaling. On first glance, this is counterintuitive, since the goal of `esync`/`fsync` was to avoid involving wineserver in wait operations at all. But this fits well into the microkernel origins from Mach, where additional functionality is provided in user-space.

Performance is good here since `mach_msg()` is fast and happens asynchronously. For a contended wait, a process needs to do 2-3 system calls (only 2 most of the time), with the waiting and signaling being directly managed by the Mach semaphore subsystem (which is around 25% faster than swapping these out with message receive for waits and message sends for signaling).

Mach semaphores have a higher creation overhead, though, which is why these are pooled client-side. Also, on the server side, several optimizations have been done to ensure that the Mach message pump runs as efficiently as possible.

Of course, a kernel extension would provide even better performance here, but these are no longer recommended by Apple, and several hurdles have been put in place to load them starting with macOS 11.0.

When a process initiates a wait with `msync`, it gets a semaphore from the semaphore pool, registers the objects now associated with this semaphore to wineserver, and starts waiting. If needed, it also unregisters it after waking again (non-successful wait or more than one wait object).

A signal operation is a Mach message for wineserver, which will look up the object and signal and destroy each of the registered semaphores associated with that object.

Races should effectively be not possible (at least I am trying to assure myself of that), since the registration and signaling happen on the same Mach port, and the message queue is supposed to be sequentially consistent across all processes. Additionally, before registration (and most likely after the client started its wait), the server rechecks the waiting condition, similar to futexes. Even if the client is preempted for a long time there and hasn't started its wait, the semaphore will be in a signaled state whenever it does.

Additionally, if available `__ulock_wait2` is used to relieve some load from the servers message handling, allowing single wait cases to have better-than-NT performance. 

## Bugs

Please report any discovered bugs. A primary aim for ```msync``` was to enhance performance over simulated ```eventfd``` objects on macOS. If ```esync``` has better performance in any situation, that's considered a bug.

For incorrect behaviour logs with at least ```+seh,+pid,+msync,+timestamp``` are much appreciated.

## Benchmarks

System: M2 Max with a CX23 based wine build.

|Test Description|msync (with ulock)|msync|esync|Server-side sync|
|---|---|---|---|---|
|Contended wait (10000000 iterations, 2 threads)|3.792806 seconds|5.891094 seconds|7.423686 seconds|&gt; 170 seconds|
|zigzag test (2 seconds timeout, 2 threads)|401605 iterations|270545 iterations|222675 iterations|60309 iterations|
|FFXIV indoors, CPU bound|219 FPS|170 FPS|145 FPS|93 FPS|

## Acknowledgements

```msync``` was inspired by ```esync```/```fsync``` and the awesome work done by Zebediah Figura on that matter.

## Contribute

Contributions to ```msync``` are welcome, be it in the form of optimizations, bug fixes, or wine rebases.

## License & Integration

```msync``` is released under the LGPL v2.1 license, and integration into other open-source projects is very welcome. If you decide to use or integrate ```msync``` into your projects, although not mandatory, a mention or acknowledgment would be appreciated.
