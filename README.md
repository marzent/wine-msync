# msync

```msync``` is a wine patch set utilizing [Mach](https://web.mit.edu/darwin/src/modules/xnu/osfmk/man/) semaphores to provide efficient NT-synchronization primitive emulation for Wine on macOS.

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

## Bugs and Limitations

Please report any discovered bugs. A primary aim for ```msync``` was to enhance performance over simulated ```eventfd``` objects on macOS. If ```esync``` has better performance in any situation, that's considered a bug.

For incorrect behaviour logs with at least ```+seh,+pid,+msync,+timestamp``` are much apprciated.

For optimal performance, ```msync``` directly calls into ```mach_msg2_trap```, meaning macOS 12.0 or a newer is required.

## Benchmarks

System: M2 Max with a CX23 based wine build.

|Test Description|msync|esync|Server-side sync|
|---|---|---|---|
|Contended wait (10000000 iterations, 2 threads)|5.891094 seconds|7.423686 seconds|&gt; 170 seconds|
|Uncontended wait (2 seconds timeout, 2 threads)|270545 iterations|222675 iterations|60309 iterations|
|FFXIV indoors, CPU bound|170 FPS|145 FPS|93 FPS|

## Acknowledgements

```msync``` was inspired by ```esync/fsync``` and the awesome work done by Zebediah Figura on that matter.

## Contribute

Contributions to ```msync``` are welcome, be it in the form of optimizations, bug fixes, or wine rebases.

