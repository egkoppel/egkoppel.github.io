---
layout: post
title: "Unwinding in a Rust kernel - part 2"
---
In the last post I managed to get unwinding working within the kernel, but there was the small issue that it would only print out addresses, and each one would need to be converted with a tool like <code>addr2line</code> to get any useful debugging information out. The best solution would probably be to get the bootloader to load in debuginfo and parse that in the same way <code>addr2line</code> does. However, this has a couple of issues:

- It would add a lot of memory overhead. According to [bloaty](https://github.com/google/bloaty), is 3-5 MiB, compared to the rest of the kernel which is only 585 KiB.
- DWARF is complicated, and I don't want to write a DWARF parser if I can come up with a good alternative.

The obvious solution then, is to look at what Linux does, and do it worse. I'm not entirely sure what Linux does for backtraces, since I've found stuff about both <code>System.map</code> and ORC debug info. The solution I went for though, was a bad version of <code>System.map</code>. According to Wikipedia, <code>System.map</code> is just the result of calling <code>nm</code> on the kernel executable, but with a slight change to add the <code>--demangle</code> argument since this is Rust, not C, and I'd like humans to be able to debug panics. Unfortunately, even with demangling, the output still ended up with unhelpful names like <code>_$LT$$RF$mut$u20$W$u20$as$u20$core..fmt..Write$GT$::write_char::hce737ea7a8f317dc</code>. Strange...

It seems the issue is down to the mangling scheme used - as of writing, <code>rustc</code> still defaults to the 'legacy' mangling scheme, rather than the new 'v0' scheme, and this trips up the demangling. Switching over to v0 mangling with<code> -C symbol-mangling-version=v0</code> seems to have fixed this, giving the much easier to read `<kernel::io::serial::SerialPort as core::fmt::Write>::write_char`. My symbol lookup algorithm also needs the symbol file to be in address order, so I also added the <code>--numeric-sort</code> argument. (Although apparently I failed to read the man page thoroughly enough the first time around and [wrote an entire executable to preprocess the output for me](https://github.com/popcorn-2/popcorn-2/blob/f27e9708fa00f90a6394b171d639f67bfa9d0db2/symbol_formatter/src/main.rs).)

The output follows a very simple format, making it very simple to parse.

```
0000000000000000 r __executable_start
ffffffff8003ccb0 t kernel::panicking::panicking
ffffffff8003ccc0 t __rust_alloc_error_handler
ffffffff8003ccd0 t core::ptr::drop_in_place::<core::option::Option<utils::handoff::Framebuffer>>
ffffffff8003cce0 t core::ptr::drop_in_place::<alloc::vec::Vec<utils::handoff::MemoryMapEntry>>
ffffffff8003cd00 t core::ptr::drop_in_place::<utils::handoff::Memory> ffffffff8003cd20 t core::ptr::drop_in_place::<u32>
ffffffff8003cd30 t <u32 as core::fmt::Debug>::fmt
ffffffff8003cd80 t <usize as core::fmt::Debug>::fmt
```

Each line starts with a symbol address in hexadecimal, padded up to 16 characters. Then a single character for the type[^1] of symbol, followed by the demangled name. Even better, on an unoptimized build, the symbol file is 600 KiB, and an optimized build gets it down to 190 KiB. It does lose line numbers, but it's still so much smaller than the DWARF debuginfo (over 20x smaller!), and so much easier to parse.

# Putting it to use

The first step was to somehow get this data into the kernel. I added a line to the bootloader to read the file <code>EFI/POPCORN/symbols.map</code> into memory if it exists, and pass it along to the kernel. (Eventually this should probably come from the actual root partition for the OS, but I haven't yet got to that point.) With access to the symbol map from within the kernel, I can now add my terrible symbol lookup algorithm into the <code>stack_trace</code> function from last post. In theory it shouldn't matter too much how slow the lookup algorithm is, since technically it shouldn't ever run\* (\*mileage may very depeding on the quality of your code). First I wrote a small parser to take in the symbol map data, and return an iterator over <code>(address, name)</code> pairs.

```rust
struct SymbolMapIterator {
	index: usize,
	data: &'static [u8]
}

impl Iterator for SymbolMapIterator {
	type Item = (usize, &'static str);
	
	fn next(&mut self) -> Option<Self::Item> {
		let original_idx = self.index;
		if original_idx == self.data.len() { return None; }
		
		let mut idx = original_idx;
		while self.data[idx] != b'\n' { idx += 1; }
		
		let data = core::str::from_utf8(&self.data[original_idx..idx]).ok()?; 
		let addr = &data[0..16];
		let name = &data[19..];
		let addr = usize::from_str_radix(addr, 16).ok()?;
		
		self.index = idx + 1;
		
		Some((addr, name))
	}
}
```

Since this will be called during a panic, it doesn't really help much to panic in the case that the symbol map is unparsable, so I just return <code>None</code>, resulting in an `<unknown>` printout in the backtrace.

Then during the backtrace, I can iterate over the symbol map to try and find the name corresponding to each function. The addresses returned during the backtrace aren't necessarily the start of the function, but are instead are the return address from each function call. This meas that I can't just do a basic equality test, and instead have to find the symbol corresponding to the closest preceding address (hence why I wanted the symbol map to be sorted).

With that in place, the kernel now spits out a lovely backtrace (the `<unknown>` symbol at the end being the main function of the bootloader)

```
kernel panicked at 'not yet implemented: Higher than 4K alignment', kernel/src/memory/watermark_allocator.rs:250:34

1: 0xffffffff80079021 - unwinding::unwinder::with_context::<unwinding::abi::UnwindReasonCode, unwinding::unwinder::_Unwind_RaiseException::{closure#0}>
2: 0xffffffff8007a911 - unwinding::unwinder::_Unwind_Backtrace::{closure#0}
3: 0xffffffff8006d498 - kernel::panicking::stack_trace
4: 0xffffffff8006db16 - kernel::panicking::do_panic_with
5: 0xffffffff8006dae2 - kernel::panicking::do_panic
6: 0xffffffff80068346 - rust_begin_unwind
7: 0xffffffff800f666c - core::panicking::panic_fmt
8: 0xffffffff80068b4d - <kernel::memory::watermark_allocator::WatermarkAllocatorInner>::allocate_contiguous
9: 0xffffffff80068a0a - <kernel::memory::watermark_allocator::WatermarkAllocator as kernel::memory::Allocator>::allocate_contiguous_aligned
10: 0xffffffff8006a503 - <kernel::memory::watermark_allocator::WatermarkAllocator as kernel::memory::Allocator>::try_allocate_contiguous_aligned
11: 0xffffffff800680d2 - kernel::kmain
12: 0xffffffff80067a45 - _start
13: 0x5cf1247 - <unknown>

FATAL: aborting
```

[^1]: The likely symbol types in this case are
	```
	A - absolute
	B - zeroed data
	D - read/write data (static variables)
	R - read only data (strings, among other objects)
	T - executable code
	```
	Each type can be either capitalised or lowercase, indicating its visibility. Capital symbols are exported and visible to code linked to the kernel, whereas lowercase symbols are only visible within the kernel.