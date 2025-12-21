---
layout: post
title: Writing a UEFI bootloader - part 1
---
For some reason, I decided that for this attempt, it would be fun trying to write a bootloader myself rather then relying on GRUB.
I could make up probably-untrue excuse about easier access to hardware control, like the [GOP](https://wiki.osdev.org/GOP), but the real reason is likely closer to "why not?".
I discovered that there's a [uefi-rs](https://crates.io/crates/uefi) crate already available, so started by reorganising the Popcorn2 directory structure to have both a kernel and a bootloader binary within one cargo workspace.
Copying the tutorial was supposed to give a nice "Hello world!", but instead it gave some lovely compiler errors about not being able chunks of the stdlib.
It seems that, like with building the main kernel, it needs `build-std` to compile the `core` crate (which wasn't explained in the tutorial).
With that out of the way, creating an EFI partition and booting qemu gives the nice "Hello world!" output, but I'd prefer to try and do something fancier.

**UPDATE (2023-11-05)**: It seems I skipped over the correct way to build this, which is actually to install the `x86_64-unknown-uefi` target through `rustup`, which provides a prebuild standard library.

<!-- <figure class="wp-block-image aligncenter size-large"><img src="https://eliyahu.co.uk/wp-content/uploads/2023/05/image-1-1024x691.png" alt="A screenshot of a qemu window, with a TianoCore logo in the centre, and the words &quot;Hello world!&quot; in the top left" class="wp-image-64"/></figure> --> 

# The GOP

The GOP (no not that one, this is the Graphics Output Protocol) is UEFI's way of drawing graphics, similar to the VESA calls when booting from the BIOS, and in theory should allow us to access the framebuffer directly.

Everything in UEFI is controlled through handles, and the GOP is no exception.
So the first step then is to actually get hold of a handle for the GOP, which is done through the logically named `get_handle_for_protocol()` function, and takes `GraphicsOutput`as a generic parameter to tell it what kind of handle we want.
Testing this out in qemu shows that it was successfully able to find the handle, so now we actually need to do something with it.

As far as I can understand from the `uefi-rs` documentation, the only useful thing to do with a `Handle` is to open a protocol with it - in this case, the `GraphicsOutput` protocol.

```rust
let Ok(gop_handle) = services.get_handle_for_protocol::<GraphicsOutput>() else { 
	panic!("Unable to locate GOP handle");
};
info!("Located GOP handle");
let Ok(gop) = services.open_protocol_exclusive::<GraphicsOutput>(gop_handle) else {
	panic!("Unable to open GOP");
};
info!("Opened GOP protocol");
```

And unfortunately running this produces neither "Unable to open GOP" nor "Opened GOP protocol".
Manually causing a panic does work properly, which means that something in `open_protocol_exclusive` is causing the system to hang.
After some digging, I found [this comment](https://github.com/rust-osdev/uefi-rs/blob/1b969948b188e74f0f98b50c9865ea9dec852bad/uefi-test-runner/src/proto/console/gop.rs#L18) in the uefi-rs test sources[^1] - I'm not sure whether this applies here too, but following the advice there does seem to have fixed the issues.
In theory now we can access the raw framebuffer, so let's try painting the whole screen red.
First I'll try pulling out some basic info, like the pixel format and resolution, which seems to work.

<!--<figure class="wp-block-image size-large"><img src="https://eliyahu.co.uk/wp-content/uploads/2023/05/image-2-1024x175.png" alt="A screenshot from qemu, with the text &quot;Framebuffer format is BGR&quot; followed by &quot;Framebuffer is 1280x800&quot;" class="wp-image-68"/></figure>--> <!-- /wp:image -->

# Drawing to the framebuffer

The framebuffer can be accessed as a raw pointer, which is how I'll draw to it for now - in the next post or two I'll probably make a safe wrapper around it, but for testing its easier to have a small `unsafe` block.

The framebuffer format that qemu gave me was BGR, with the last byte of each 32 bit pixel reserved, so one would naturally assume the correct value for red is `0x0000ff00`. Except its not.
Logically.

x86, like most architectures, is little endian.
This means that even though the individual bit order within a byte is stored as you'd expect, the order of bytes are reversed.
This does have some advantages, for example being able to cast between differently sized types without any pointer arithmetic, since the pointer always points to the least significant byte.
Unfortunately for this, it means that `0x0000ff00` gets written into memory as `00 ff 00 00`, giving green, not red as one would've hoped.
Accounting for this, the correct value for red is `0x00ff0000`, which does work properly.
Weirdly this means that BGR ends up being written in RGB, and RGB is written as BGR.

---

In the next post I'll try and get filesystem access working, and hopefully set something up to draw some text and potentially basic BMP files onto the screen.
From there I can hopefully build some kind of menu with user input, and maybe try loading the kernel and handing off to that.

[^1]: From that comment, I think the problem is that the GOP protocol is already opened by the uefi-rs helpers so it can provide output through the log crate. When trying to open the protocol in exclusive mode, the UEFI firmware is supposed to shut down other applications that also have exclusive access to the protocol, which I think in this case includes our bootloader itself.