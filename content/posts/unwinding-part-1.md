---
title: "Unwinding in a Rust kernel - part 1"
date: 2023-08-01
draft: false
series:
  - "Rust kernel unwinding"
---

Apparently it's been nearly two months since the previous post. Not to worry though, because (after A levels finished) I've been busy with more Popcorn, and made significant progress in both the design and implentation. There's a couple of UEFI posts still to come (one currently in the pipeline) but I thought it might be worth posting soemthing in the meantime to give the appearance that this blog is not in fact dead.

As I've been writing some of the kernel submodules (and finding too many bugs in them) I realised that it might be time to start writing unit tests. To properly test everything, it would make sense to be able to run the tests on representative hardware, and so it would be nice if the test executable produced by `cargo` could be run in the same way as the kernel is.

To get started with that, I decided to follow the [testing section of Phillip Opperman’s great blog series](https://os.phil-opp.com/testing/). Since I'm rolling my own bootloader rather than using `bootimage`, I unfortunately haven't yet set up `cargo test` to actually run the tests itself, but I have a few other fancy features to make up for it.

> For example, the `#[should_panic]` attribute relies on stack unwinding to catch the panics, which we disabled for our kernel.
{cite="Phillip Opperman's blog"}

This bothered me. Just disabling features is no fun. There isn't any challenge in that. On top of the lack of `#[should_panic]` we also lose a couple of another nice features - (a) tests stop running as soon as one fails, since there's no way to recover from the panic, and (b) there's no way to get a nice backtrace on any panic, making debugging slightly more difficult/annoying. So I set about getting stack unwinding to work in the kernel. I managed to get it working for C++ in Popcorn1, so it shouldn't be <em>too</em> bad, right?

I quickly discovered the [unwinding crate](https://crates.io/crates/unwinding) which sounded like exactly what I wanted - even better, it literally has a bare metal section in the docs! Unfortunately, it was (very slightly) too good to be true - `lld` seemed to be refusing to link `__GNU_EH_FRAME_HDR`, even with the `--eh-frame-hdr` option passed. And the same issues happened with `__etext` as well. So I switched to providing the symbols myself, which wasn't too difficult. First I added a `GNU_EH_FRAME` section (I'm not sure this is necessary but the normal linker script does so why not copy it), as well as a `.eh_frame_hdr` and `.eh_frame` segment.
{{< sidenote >}}This is a magic symbol inserted by GNU-like linkers to tell unwind implementations where the unwind information is stored - it is equivalent to the `__eh_frame` symbol manually added later on{{</ sidenote >}}

```linker-script
PHDRS {
	realmode PT_LOAD FLAGS(0x010007) ;
	rodata PT_LOAD FLAGS(0x4) ;
	text PT_LOAD FLAGS(0x5) ;
	data PT_LOAD FLAGS(0x6) ;
	dynamic PT_DYNAMIC ;
	gnu_eh_frame 0x6474E550 FLAGS(0x4) ; /* NEW */
}

SECTIONS {
	/* ... */
	.eh_frame_hdr : {
		*(.eh_frame_hdr .eh_frame_hdr.*)
	} :gnu_eh_frame :rodata
	. = ALIGN(8);
	PROVIDE(__eh_frame = .);
	.eh_frame : { *(.eh_frame .eh_frame.*) } :rodata
	
	. = ALIGN(4K);
	.text : {
		*(.text .text.*)
		PROVIDE(__etext = .);
	} :text
}
```

With those added, everything now linked perfectly. Time to start work on the actual unwinding. The unwinding library can provide a panic handler through either the `panic` or `panic-handler` features. However, this would either add a dependency on libc, or not give us the nice error messages and backtrace. So the obvious step is to look at what they actually enable, and <s>write a better version</s> copy it. Originally my panic handler looked like this:

```rust
#[panic_handler]
fn panic_handler(info: &PanicInfo) -> ! {
	sprintln!("{info}");
	loop {}
}
```

However, now we want to make it print a backtrace, make sure it isn't already panicking (for example, if a `Drop` implementation panicked we don't want to go into a recursive loop), and then begin unwinding. To keep track of recursive panics, we can keep a global counter of the number of panics already happening. For now, I'm using an `AtomicUsize`, but if the kernel becomes multithreaded (currently not sure if that actually makes sense) then we want the panic counter to be thread local, since we don't want one thread to give up panicking because a different thread is in the middle of unwinding.

If it's ok to continue with the unwind, then we can call into `unwinding::panic::begin_panic()` to start the actual unwinding process. Each panic includes a payload (as can also be seen in [`std::panic::panic_any()`](https://doc.rust-lang.org/std/panic/fn.panic_any.html)) - in the case of a normal panic this is just an empty struct (I think - at least in the unwinding crate it is, but I haven't looked through the standard library implementation). The payload does need to be stored on the heap, since the unwinder is allowed to trash the stack during the unwind process, destroying the payload if it was stored on the stack. Interestingly, `begin_panic()` isn't marked as not returning, because, yes, it can fail at failing (eg. due to a corrupted stack). If it fails to fail, then the best solution is probably to print an error and abort (or in the case of this kernel, `loop {}`). This ends up with a panic handler looking something like

```rust
#[panic_handler]
fn panic_handler(info: &PanicInfo) -> ! {
	sprintln!("{info}");
	
	struct NoPayload;
	do_panic_with(Box::new(NoPayload))
}

fn do_panic_with(payload: Box<dyn Any + Send>) -> ! {
	#[cfg(panic = "unwind")] {
		if PANIC_COUNT.compare_exchange(0, 1, Ordering::Acquire, Ordering::Relaxed).is_err() {
			// PANIC_COUNT not at 1
			// already unwinding
			sprintln!("FATAL: kernel panicked while processing panic.");
			loop {}
		} else {
			// new unwind
			let code = unwinding::panic::begin_panic(payload);
			sprintln!("FATAL: failed at failing, error code {}.", code.0);
			loop {}
		}
	}
	#[cfg(not(panic = "unwind"))] loop {}
}
```

Preferably we also want a stack trace. Luckily, libunwind (the ABI that the unwinding crate conforms to) has an easy way to do that (and yes - credit again to the author of unwinding who I copied this code from). `_Unwind_Backtrace` takes a callback function and a pointer to some state data, and calls the callback for each stack frame in the backtrace.

```rust
fn stack_trace() {
	use unwinding::abi::{UnwindContext, UnwindReasonCode, _Unwind_GetIP, _Unwind_Backtrace};
	use core::ffi::c_void;
	
	extern "C" fn callback(
		unwind_ctx: &mut UnwindContext<'_>,
		_: *mut c_void,
	) -> UnwindReasonCode {
		sprintln!("{:#18x}", _Unwind_GetIP(unwind_ctx));
		UnwindReasonCode::NO_REASON
	}
	
	_Unwind_Backtrace(callback, core::ptr::null_mut());
}
```

For each frame in the stack, print the instruction address returned by `_Unwind_GetIP()` - this is actually the instruction one after each `call` instruction, since that's the address where each function will return to. At some point later I might look into converting these into function names, but for now I'll have to stick with using the useful `addr2line` tool.

There's also one last function we need to achieve the original goal of `#[should_panic]` tests - `catch_unwind()`. The unwinding crate does provide this, but using it directly will cause a small bug. Remember how we have a counter of the number of current panics? Well that never gets decremeted when we catch a panic, and so, as soon as we catch a panic and then later panic again, our code would think it's still unwinding from the first (caught) panic. It's an easy fix though - just decrement the panic count in a wrapper function:
```rust
pub fn catch_unwind<R, F: FnOnce() -> R>(f: F) -> Result<R, Box<dyn Any + Send>> {
	use unwinding::panic::catch_unwind as catch_unwind_impl;
	
	let res = catch_unwind_impl(f);
	PANIC_COUNT.store(0, Ordering::Relaxed);
	res
}
```

# Finally, the tests

The first step is to wrap our test functions in a `catch_unwind()`, allowing us to continue with the rest of the tests even after one fails. We can also add some other extra niceties, like returning the status of the test so we can print a summary at the end. This results in the `Testable` implementation (originally from Phillip Opperman's post) looking like

```rust
impl<T> Testable for T where T: Fn() {
	fn run(&self) -> Result {
		sprint!("{}...\t", core::any::type_name::<T>());
		
		match panicking::catch_unwind(self) {
			Ok(_) => {
				sprintln!("[ok]");
				Result::Success
			},
			Err(_) => {
				sprintln!("[FAIL]");
				Result::Fail
			}
		}
	}
}
```

Then we can implement `#[should_panic]`. With the `#[should_panic]` attribute, what we want to do is essentially invert the function it's applied to - if the function panics, we want to return normally, and if the function returns normally, we want to panic. To make this happen, I wrote a small procedural macro which wraps the function in a new function of the same name (to keep the user visible name of the test the same) and calls the original function through `catch_unwind()`:

```rust
#[proc_macro_attribute]
pub fn test_should_panic(_attr: TokenStream, item: TokenStream) -> TokenStream {
	let func = parse_macro_input!(item as ItemFn);
	let ident = func.sig.ident.clone();
	let output = quote!{
		#[test_case]
		fn #ident () {
			#func
			
			match crate::panicking::catch_unwind(#ident) {
				Ok(_) => panic!("Test did not panic"),
				Err(_) => {}
			}
		}
	};
	output.into()
}
```

I would have liked to keep it as close to the standard library implementation as possible. However, currently it seems there's a <a rel="noreferrer noopener" href="https://github.com/rust-lang/rust/issues/100263" target="_blank">bug</a> with the `#[test_case]` macro (and a yet-to-be-merged fix), preventing it from being applied with the custom `#[should_panic]` macro, and so instead it gets applied by the macro instead. It also seems that even without using the stdlib test framework `#[should_panic]` is still a reserved attribute, and so I've had to change the name of mine slightly. But with that macro in place, we have unwinding working with a backtrace, and can write tests that should panic, just as I originally wanted.

# Finishing off the interface

The Rust stdlib panic interface has a couple extra features that we don't have as of yet, and it would be nice to give a complete interface. The two that seem most pertinent are `std::thread::panicking()` which checks if the current thread is panicking. This can be used in some `Drop` implementations, for example in the implementation of `MutexGuard`, to "poison" the object. In the case of `Mutex`, a guard would normally be held while updating the inner object. If the updating function panics partway through, then it's possible the object contained in the mutex is left in an invalid, partially updated state that other code shouldn't see. To prevent other code from seeing this invalid state, the mutex becomes "poisoned", setting an internal flag that requires callers to manually override the poisoning to be able to lock the mutex agin. This is done by checking the result of `panicking()` within `Drop` to decide whether to just unlock the mutex or whether to poison it too. Our implementation can be very simple - just check the value of our panic counter.

The second function is `std::panic::resume_unwind()`. This allows rethrowing a panic after catching it and doing some handling with `catch_unwind()` - equivalent to C++'s

```c++
try { /* ... */ }
catch (std::exception& e) {
	/* ... */
	throw;
}
```

This is very easy to implement since it's exactly the same function as the `do_panic_with()` function from earlier.

---

Hopefully this has been a useful interlude between more kernel specific posts, and if not, then at least interesting. I'm not sure how long it'll be until the next UEFI post, but I'll aim for less than two months this time. 

Code, as usual, is on [GitHub](https://github.com/popcorn-2/popcorn-2).
