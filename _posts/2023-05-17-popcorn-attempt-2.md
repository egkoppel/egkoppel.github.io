---
layout: post
title: "Popcorn - attempt 2"
---
Having been demotivated from continuing with Popcorn attempt 1 with all the strange, hard to track down memory bugs, I decided to start again (and write this series of blog posts alongside) but in Rust.
I technically have already done some OS development in Rust, in the experimental pre-Popcorn times, but at that point had never used it before, and didn't properly understand how to write idiomatic Rust.
This caused a lot of borrow checker fighting and unnecessary `unsafe` blocks, which is why I made the decision to write attempt 1 of Popcorn in C++ instead.

A lot of the code for Popcorn 2 will probably come from porting and modifying Popcorn 1 (and I'll probably leave Popcorn 1 on [GitHub](https://github.com/egkoppel/popcorn-1) in an archive), but with changes where necessary to more fit in with Rust paradigms.
A fair amount of code can be safely ignored - since I was trying to fully utilize modern C++ paradigms, I had gone to some work porting chunks `libc++` (for STL types) and `libcxxrt` (for C++ exception handling), which can all be replaced with Rust's own container and `Result` types.

Like with Popcorn 1, I'll initially only be targeting `amd64`, but depending on how far this gets, might look into ports to other architectures later.
To aid with this, I'll be trying to keep architecture specific code in separate modules.

---

The first step then is to get `cargo` up and running, with a custom build script to assemble the bootstrap code, and link it to the rest of the Rust code.
Calling into `nasm` was easy enough - just loop through file paths, generate an output filename, then just call `nasm` with the right arguments.

```rust
for file in asm_files {
	println!("cargo:rerun-if-changed={file}");
	
	let output_file = Path::new(file).file_name().unwrap().to_str().unwrap();
	let output_file = format!("{out_dir}/{output_file}.o");
	let status = Command::new("nasm")
		.args([file, "-o", &output_file, "-f", "elf64", "-F", "dwarf", "-g"])
		.output()
		.unwrap();
	if !status.status.success() {
		panic!(
			"Failed to compile {file}:\n{}",
			String::from_utf8_lossy(&status.stderr)
		)
	}
}
```

I do plan to try and add support for `jobserver` in the future, with parallel assembly, but I think this is a good first start. Now onto the linking...

---

Linking the built files was fine - adding a call to print `cargo:rustc-link-arg={output_file}` on each file worked perfectly. Persuading `rustc` to use the kernel linker script however...

In theory printing `cargo:rustc-link-arg=-T {linker_script}` should work fine - and it does correctly pass the arguments onto `lld`.
Which then promptly complains the file doesn't exist.
Relative file path issues?
Nope - passing the absolute file path also fails with the same error.
Yet copying the file path from the output of `rustc` and trying to `cat` it works perfectly fine.

\[A few hours later]

There was an extra space.
Between the `-T` in the linker argument and the path.
Apparently `lld` decided that meant the path had a space before it.
That has now been fixed, and the asm all links together with the Rust.

---

Overall, after sorting out the linker issues, I've found setting the project up in Rust much simpler than getting CMake up and running - CMake seems vastly more configurable, but cargo's `build.rs` seems adaquate for this so far.
Also, the `build-std` feature of cargo makes getting a modern stdlib much simpler than having to try and port or cross-compile `libc++`.


Code for Popcorn 2 is on [GitHub](https://github.com/egkoppel/popcorn-2).