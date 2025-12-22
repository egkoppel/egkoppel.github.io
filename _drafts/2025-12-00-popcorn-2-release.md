Source code (and maybe prebuilt ISOs?) is all available on [GitHub](https://github.com/popcorn-2/popcorn-2).

---

It's been a very long time since I posted anything here, mainly cause I just haven't had the motivation to write anything interesting. Many eons I started (and as is the way, never finished) a series of posts about writing a UEFI bootloader in Rust. That was all part of my long-running side project - Popcorn2 (the 2 because i wrote Popcorn[^1] even longer ago in C++ and, having messed around with Rust for a bit before, I ended up getting frustrated at C++ move semantics and type system and page faults and a bunch of stuff, so I did what any insane person would do and rewrote it in Rust).

To put it succinctly, Popcorn2 (which I'll probably just refer to as Popcorn because it's faster to type and less clunky to say) is a microkernel written almost entirely by me, because I have ~~brilliant~~ probably terrible ideas about how an OS "should be designed" which I wanted to experiment with. Many of the design choices are probably very opinionated, and based on what I think is as close to a pleasant design as I can get.

Despite having spent over two years (on and off) on this revision of Popcorn, with school and subsequently uni taking up time, it's not _that_ far along. So what can it do? At the moment, it boots in QEMU, starts a userspace and begins launching servers and drivers. It has enough drivers and enough of `libc` in place that it can run Pico-8 Celeste (because I absolutely love Celeste and have never touched Doom) - basically a Lua runtime plus a custom C emulator to tie the Lua code into the Popcorn window manager - and can run a few bits of `uutils` (the rest requiring a bunch of patching of dependencies).

Much of the code is very hacked together and not in the best state either structure or documentation-wise, but it does function. Additionally, in an attempt to audit the `unsafe` bits, where possible I've tested chunks under `miri`, and run the entire OS with KASAN enabled (both running without error). 

# Everything is an object!

(ok why the fuck does half the codebase refer to protocols and the other half refers to interfaces wtf is this codebase at this point)

Popcorn draws on the design philosophy of UNIX and Plan9 of "everything is a file", instead turning it into "everything is an object". Personally, I don't like the fact that file I/O has to be forced onto everything - what does it mean to call `read` or `seek` on a PCI device? And what if I need to expose a more complex interface? - then you just end up hacking everything onto an `ioctl`.

Instead I decided to draw on OO paradigms, and pulled a lot of inspiration from Wayland, dBus, RedoxOS, Fuschia, and probably some more. Instead of files, we have objects. And instead of every object supporting file I/O + `ioctl`, it can support a set of protocols - each protocol being a set of methods and enums defined in a `pip` (Popcorn IPC Protocol) file. For example, `core/io.pip` is defined as follows:

```
interface core.io.Read() @ 0000-0000-0000-000000000002 {
    func@1 read() -> [byte]
}

interface core.io.Write() @ 0000-0000-0000-000000000003 {
    func@1 write(buf: [byte]) -> uint
}

interface core.io.Seek() @ 0000-0000-0000-000000000004 {
    func@1 tell() -> uint
    func@2 set_pos(pos: uint)
}
```

This defines some of the standard file I/O methods - `read@core.io.Read()` is a method called on an object, which returns a slice of untyped bytes. `write@core.io.Write()` does the reverse - it takes a slice of untyped bytes, and returns an unsigned integer as to how many bytes were successfully written.

(probably also something about how methods are supposed to map directly to high level language constructs, and interface bindings can just be generated from the `pip` file)

Each process contains a set of handles rather than file descriptors, where each handle in a numeric descriptor referring to a specific object with a specific set of protocols supported on it[^2]. Therefore owning a handle is a capability to perform whatever methods are available on it. For example, owning a handle that implements `driver.PciDevice` gives the process full control over the PCI device, or instead just a object implementing the `mem.Pager` protocol (allows `mmap`ing into the address space) would just give access to a specific BAR.

Additionally, to allow inheritance of handles, and therefore capabilities, each process is passed a handle table at startup - a mapping of string names to handle numbers. Typically this would include handles for `io.stdin`, `io.stdout`, `io.stderr` and `thread.main`, but the parent process can add arbitrary handles to this, such as the window manager passing a `wm.window` handle or similar to graphical programs to allow them to interact with the window manager.

# Servers and handles

(this whole section probably needs cleaning up i'm tired and it's a bit of a ramble)

Something is required to be on the other end of each object, for method calls to be dispatched to, and in the case of Popcorn, these are called servers. As you might guess from the heading of the previous section, every server is just an object implementing the `core.srv.Sync` protocol[^3]! This contains three methods: `next` and `reply`, which are the pair of methods for getting the next method call to process and returning the response back to the calling process, and `forge`, a method for taking the local identifier that a server uses (eg. maybe an inode number for a filesystem driver, or the TID for the process server) and creating an `Arc<Handle>` object in the kernel - roughly equivalent to `struct file` under Linux - which can then be mapped into another process's handle map to give it access to the object. A diagram might make this clearer:

```
                            Client process handle            Arc<Handle>
   Client process             table (in kernel)              (in kernel)
                      
+-----------------+      +----+--------------------+      +-----------+---+
| write(handle 2) | -+   | Id |    Arc<Handle>     |  +-> | server_id | 3 |
+-----------------+  |   +----+--------------------+  |   | local_id  | 5 |
                     |   | 0  | 0xffffd0000002dd90 |  |   | protocols |   |
                     |   | 1  | 0xffffd0000003dd40 |  |   +-----------+---+
                     +-> | 2  | 0xffffd0000003e610 | -+
                         | 3  | 0xffffd00000064a00 |
                         +----+--------------------+
```

When the client process calls `write` on handle 2, the kernel looks up which `Arc<Handle>` is meant by "handle 2" in that process, and if the protocol called on it is available, and if so dispatches a call to server 3 to tell it that `write` was called on what that server considers "handle 5". This allows `dup` to function just by having two handle numbers map to the same `Arc<Handle>` in the client process, and means servers don't have to cooperate to avoid sharing handle IDs. The `forge` call is required to create the `Arc<Handle>` object in kernel memory such that another process can actually refer to the object a server has just created.
# System calls and IPC

This whole time I've talked about processes calling methods on objects, but I've never explained how that actually works. And in my opinion, this is one of the biggest departures from any kernel I know of. Depending on how you count, there are anywhere between 8 and 2^128 system calls. None of these system calls are any kind of "call method" or "send IPC message" - the methods defined by each protocol **are** the system calls.

Each system call has a 128 bit UID, made of a 32 bit method ID and a 96 bit protocol UID. 

- marshalling
# (Almost) everything is async!

# Kernel modules

Some very long time ago when first starting Popcorn2, I had the thought "kernel modules make sense for a microkernel because microkernels are modular". There was a whole grand plan of having different scheduler implementations and memory allocators and all that which could be swapped out to try and tune the kernel to whatever the current hardware situation is - things like using a more power efficient scheduler when switching to battery power on a laptop. This thought has kinda stuck throughout the design, but has definitely been cut back as it overcomplicated everything and ultimately seemed pretty pointless.

Modules still exist in some sense, in that the allocator implementations (physical, virtual, and heap) and eventually the scheduler and maybe hardware abstraction layer are all separate crates linked into the kernel at the end. The eventual plan is that the kernel will be distributed as a static library, first- and third-party modules can be installed as static libraries, and the kernel package has a post-install hook to link it to the installed modules, and then boot the linked executable with whatever modules are installed on the system.

To facilitate this, there are two kernel crates within the codebase - `kernel` and `kernel_api`. `kernel` contains the actual kernel logic, whereas `kernel_api` exposes effectively a stable public interface to the kernel. This includes utilities like spinlocks, but also converts a number of magic ABI symbols exported by the kernel (such as `__popcorn_kpt_map_contiguous` to map a range of pages into the kernel address space) into higher level abstractions (like `KERNEL_ADDRESS_SPACE.map_pages(...)`). This also allows the underlying kernel ABI to be completely unstable, while maintaining semver to downstream modules. Technically if the kernel ABI changes, the `kernel_api` crate could even do runtime selection between different ABIs based on a magic version symbol, allowing modules to be completely forwards compatible with kernels.

# Limitations

- no counter/rtc
- no IRQs
- very allocation heavy and unoptimized syscall/ipc path

---

This post has focused entirely on the design of the Popcorn2 kernel. In theory, any set of userspace programs could sit atop the kernel (assuming it follows the microkernel IPC everything-is-an-object philosophy) and create an OS, but given the entire premise of this project being "I want to implement my opinionated design", of course I have my own opinionated userspace too. Since this is getting a bit long for one post, if I magically gain the motivation to write more in the future, I'll go through the design of the userspace in depth too.

[^1]: The name Popcorn originally coming from me coming into school one day, declaring "I'm learning how to write a kernel - what do I call it?" at my friends, and one them responding with "kernel, like the popcorn thing?".

[^2]: Technically, a handle actually refers to multiple objects, and method calls are dispatched to a particular object based on which interface the method comes from.
	For example, (very contrived example) calling `read()` on a handle will dispatch the read method to one object, but calling `write()` on the same handle could dispatch it to a different object.

[^3]: A long time ago there was a plan for a `core.srv.Async` protocol too but because of the way everything worked out, that ends up not being needed and the name has just stuck