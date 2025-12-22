---
title: "Writing a UEFI bootloader - part 2"
layout: post
---
At the end of the last post, I said that I'd probably start working on filesystems and image rendering.
And I'm pleased to say that both of those have been a success.

<!--<figure class="wp-block-image aligncenter"><img src="https://eliyahu.co.uk/wp-content/uploads/2023/05/telegram-cloud-photo-size-4-6033023274180525859-m.jpg" alt="A screenshot of the corner of a qemu window, showing a green checkmark in a grey box" class="wp-image-84"/><figcaption class="wp-element-caption">The very first image from disk</figcaption>-->

I started writing this as a single post, but when trying to arrange it into something actually readable, realised it made more sense to split into two.
This will likely end up being fairly short, with a much longer post releasing tomorrow.

# 'Simple' File System Protocol

Ok, that might be tad disingenuous - it is actually pretty simple once you read the documentation to discover that `uefi-rs` `Path`s only accept `\` as a separator, unlike the `std` `Path`s (although looking further into this, it seems like this is due to UEFI and not the crate being lazy).

**Update (2023-06-04):** It seems the `PathBuf` type (an owned `Path`) does automatic separator conversion when creating from a `CString16`

The easiest way to start is to use the same filesystem that the bootloader was loaded from.
We know it definitely exists, and furthermore, we know it's FAT32, which I think is a requirement for the Simple File System Protocol.
We also don't need to worry about searching through connected disks to find it, since the `Handle` passed as the first parameter to `main()` lets us find it easily.
Calling `BootServices::get_image_file_system()` and passing the `Handle` opens the file protocol for us, and gives access to the root directory on the EFI partition.

---

## An aside: UEFI character encoding

For some reason, UEFI uses UCS-2 internally for all its character encoding (which is slightly different to UTF-16, the former using fixed width 16-bit codepoints, and the latter using either 16 or 32 bit characters to address the same codepoint space as UTF-8 can), whereas Rust uses UTF-8 for all string literals.
On top of that, UEFI also expects all strings to be null terminated, like C does.
Unfortunately this means that we can't use the native `str` and `String`[^1] types, and instead have to use the `uefi-rs` provided `CStr16` and `CString16` types.
To convert string literals, we can wrap each literal in the `cstr16!()` macro.

---

Once we have the filesystem object, `uefi-rs` provides a nice wrapper API, called `FileSystem`, around the underlying UEFI Simple File System Protocol.
I placed a `text.txt` file in `/EFI` on the EFI partition qemu used.
Then we can easily read it into a `Vec<u8>` with the `read()` function.
One small difference to `std::fs` here is that, while both the `std` and `uefi` `read()`s take anything that can be borrowed as a `Path`, `CStr16` (unlike `str`) can't be implicitly borrowed as a `Path`[^2]. This means we need to call `Path::new()` on the string to convert it. (However, this is just some pointer casting and so just a no-op.)
With the file now in a `Vec<u8>`, we can convert it to a `str` (assuming the file is UTF-8), and print it out to the screen[^3].

The other way to read files is closer to `std::fs` APIs (where the caller provides the buffer to read into), and is probably the more suitable method when loading the kernel image later, since we want to be able to control alignment and location of the loaded kernel code.
It also doesn't need memory allocation, and so can be used in a bootloader without a heap.
Like the GOP, it requires us getting a handle for the protocol, and then opening the protocol.

```rust
let Ok(fs_handle) = services.get_handle_for_protocol::<SimpleFileSystem>() else {
	panic!("Unable to locate FS handle");
};
let fs = services.open_protocol_exclusive::<SimpleFileSystem>(fs_handle);
let Ok(mut fs) = fs else { panic!("FS protocol not opened") };
```

Like with opening the GOP, if we use `open_protocol_exclusive()` we need to make sure the rest of our bootloader doesn't already have it open.
If you've used the `FileSystem` API elsewhere, you need to make sure it's been `drop()`-ed first, otherwise exclusively opening the protocol will fail[^4].
To get the root directory of the volume, there's a `open_volume()` function on the filesystem object.
Since the medium could have been removed between opening the protocol and opening the volume, or the filesystem could be corrupted, this can error.
If it opens successfully, then we can call `read()`, passing in the path (but this time taking a raw `CStr16`), the mode (likely `FileMode::Read`) and file attributes (which are ignored unless creating a file, so can be left as `FileAttribute::empty()`).
The `open()` function however, doesn't distinguish between files and directories, so to actually read it we need to first convert it into a `RegularFile` using the `into_regular_file()`.
Then we can finally call `read()`, passing in the buffer to read into.

---

With file access in place, we're now well on the way to a functional bootloader.
In theory you could go straight from here to loading a kernel, but I'm going to take a detour into graphics and user input in the next few posts, to try and make a more configurable bootloader.

[^1]: For the non-Rust programmers among you, `String` is a heap allocated string buffer (similar to a `std::string` in C++). `str`, on the other hand, is a slice of a string - it has no fixed size, so can't be created as a type on its own, only as a reference which contains both a pointer to the beginning of the string as well as the length (similar to a `std::string_view` in C++).

[^2]: This is because `CStr16` doesn't implement the `AsRef<Path>` trait, which allows borrowing objects as a reference to different type than they are.

[^3]: If you've been following along with the other posts in this series, then you may have your own framebuffer already set up. If you haven't adjusted any of the GOP settings (like switching resolution) then the logging functions provided by uefi-rs still work fine (and just overwrite sections of the framebuffer). However, if that no longer works, then you'll need to set up some other output system, like a font renderer (which I'll probably make a post about soon).

[^4]: This returns a Simple File System protocol for the EFI partition - there should be a way to open a protocol for other partitions, but I haven't yet figured this out.