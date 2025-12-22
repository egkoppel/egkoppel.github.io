---
layout: post
title: "Writing a UEFI bootloader - part 4"
---

It's been a while since I've written a post in this series, but not to worry, because stuff has been happening.
The bootloader has significantly progressed since the last post, I just haven't had the want or time to write another blog post until now.
So the big change for this post is that the UI is now actually somewhat usable.
I tried for a bit of time to write a UI library which went nowhere since I couldn't wrap my brain around it.
For a while I had given up on that idea and was planning on this post coming much later in the series, until I realised while writing a project for work that [LVGL](http://lvgl.io) exists - so now I have LVGL running in the bootloader.
The UI could probably do with improving, but the actual code is there.
I was originally planning to use [lvgl-rs](https://github.com/lvgl/lv_binding_rust) which seem to be the official LVGL rust bindings.
Unfortunately while messing around in it I discovered what I think are some soundness issues (currently writing a bug report).
I had a go at trying to patch them, but couldn't figure out how to do it in the current design without propagating generics throughout half the codebase in ways that didn't make sense. 
Instead I started working on my own version (currently unpublished) based on a lot of the original code but with some changes to the architecture. Some work later, and I've ended up with this.

<!--<figure class="wp-block-image aligncenter size-large"><img src="https://eliyahu.co.uk/wp-content/uploads/2023/10/screenshot-1024x615.png" alt="" class="wp-image-228"/></figure>-->

The first step is improving the framebuffer design.
Under the existing system, flushing to the screen ends up being slow because we're directly copying into VRAM manually.
However, the GOP also supports hardware bit-blitting, where the GPU firmware will automatically copy some portion of memory into the framebuffer, including copying sections of an in-memory buffer to a rectangular section of framebuffer.
This makes it extremely easy to implement the LVGL display flush function, since it's expecting exactly that.

```rust
pub struct Gui;
impl Gui {
	fn flush_display(gop: &mut GraphicsOutput, update: DisplayUpdate) {
		let update_width = update.area.x2 - update.area.x1 + 1;
		let update_height = update.area.y2 - update.area.y1 + 1;
		
		for c in update.colors.iter_mut() { c.set_a(0); }
		
		let buffer: &[BltPixel] = unsafe {
			// SAFETY: alpha channel has been set to 0 since reserved under UEFI
			// memory layout of BltPixel and LVGL Color is identical
			mem::transmute(update.colors)
		};
		
		let blt_op = BltOp::BufferToVideo {
			buffer,
			src: BltRegion::Full,
			dest: (
				update.area.x1.try_into().unwrap(),
				update.area.y1.try_into().unwrap()
			),
			dims: (
				update_width.try_into().unwrap(),
				update_height.try_into().unwrap()
			),
		};
		
		gop.blt(blt_op).expect("Failed to flush display");
	}
}
```

First we calculate the width and height of the updated section - LVGL passes the coordinates of the top-left and bottom-right corners, whereas UEFI expects the width and height of the area to draw to.
Then we need to convert the colour types - they are the same 32 bit RGB layout in memory so a `transmute` is safe.
I can't find any specific requirements in the UEFI specification for the value of the reserved final byte, so I'm setting it to zero to err on the cautious side.
With the colour format converted, we can put together a Bit Blit operation to pass off to UEFI - we want to copy from a memory buffer to the screen, so we use a `BufferToVideo` operation (operations also exist for the reverse, as well as filling an area with solid colour, and copying a section of the screen to somewhere else on the screen, which is useful for implementing scrolling).

Trying to build this however gives us an error from LVGL.
<code><span style="color: #fc5e53; font-weight: bold;">error</span>: failed to run custom build command for `lvgl-sys v0.6.2` Caused by: process didn't exit successfully: `target/debug/build/lvgl-sys-c57633816a573ab0/build-script-build` (exit status: 101) --- stderr thread 'main' panicked at .cargo/git/checkouts/lv_binding_rust-d86feb7597e107b7/a18027c/lvgl-sys/build.rs:73:25: The environment variable DEP_LV_CONFIG_PATH is required to be defined note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace</code>

This is because LVGL expects to read its own configuration from a C header file, and it needs to know where to locate that.
The Rust wrapper crate expects to find the path to the config folder in an environment variable called `DEP_LV_CONFIG_PATH`, so we need to pass that into `cargo`.
In the `.cargo/config.toml` file in the root of the bootloader crate, we can add custom environment variables that `cargo` will pass on.
The `relative` key means that `cargo` will automatically prefix the value with the path to our crate, and therefore the `value` needs to be the name of a folder inside the crate containing an `lv_conf.h` file[^1].

```toml
[env] DEP_LV_CONFIG_PATH = { relative = true, value = "lvgl" }
```

Now we can get on with setting up LVGL.
First we need to call LVGL's `init` function, then create a `DrawBuffer`, display driver, and finally register the display driver with LVGL to create a `Display`.
The `DrawBuffer` is a buffer that LVGL uses to composite into, before calling our provided flush function to paint the composited buffer onto the screen.
LVGL can also use a technique called double buffering, where while it's painting to the screen from one of the two buffers, it can already start compositing the next frame into a second buffer, before swapping the two buffers around.
However, in a bootloader we aren't really going for framerate so there isn't much point adding the extra complexity, and we also don't have a way to flush the display in the background, since that requires something like DMA.
A first implementation might look a little something like this.

```rust
pub fn new(gop: &mut GraphicsOutput) -> Self {
	let (width, height) = gop.current_mode_info().resolution();
	lvgl2::init();
	
	let mut buffer = DrawBuffer::new(8000);
	let mut driver = Driver::new(
		&mut buffer,
		width,
		height,
		|update| Self::flush_display(gop, update)
	);
	let display = Display::new(&mut driver);
	
	Self { display }
}
```

Unfortunately, this gives us a couple of borrow checker errors.

<code style="white-space: pre;">
<span style="color: #fc5e53; font-weight: bold;">error[E0515]</span>: cannot return value referencing local variable `driver`
<span style="color: #6eb5fa; font-weight: bold;"> --></span> bootloader/src/framebuffer/mod.rs:16:2
<span style="color: #6eb5fa; font-weight: bold;">   |</span>
<span style="color: #6eb5fa; font-weight: bold;">12 |</span> let display = Display::new(&mut driver);
<span style="color: #6eb5fa; font-weight: bold;">   |</span>                            <span style="color: #6eb5fa; font-weight: bold;">----------- `driver` is borrowed here</span>
<span style="color: #6eb5fa; font-weight: bold;">13 |</span>
<span style="color: #6eb5fa; font-weight: bold;">14 |</span> Self { display  }
<span style="color: #6eb5fa; font-weight: bold;">   |</span>        <span style="color: #fc5e53; font-weight: bold;">^^^^^^^ returns a value referencing data owned by the current function</span><br>
<span style="color: #fc5e53; font-weight: bold;">error[E0515]</span>: cannot return value referencing local variable `buffer`
<span style="color: #6eb5fa; font-weight: bold;"> --></span> bootloader/src/framebuffer/mod.rs:16:2
<span style="color: #6eb5fa; font-weight: bold;">   |</span>
<span style="color: #6eb5fa; font-weight: bold;"> 7 |</span> &mut buffer,
<span style="color: #6eb5fa; font-weight: bold;">   | ----------- `buffer` is borrowed here</span>
<span style="color: #6eb5fa; font-weight: bold;">...</span>
<span style="color: #6eb5fa; font-weight: bold;">14 |</span> Self { display  }
<span style="color: #6eb5fa; font-weight: bold;">   |</span>        <span style="color: #fc5e53; font-weight: bold;">^^^^^^^ returns a value referencing data owned by the current function</span></code>

The issue here is that the display internally stores a reference to its corresponding driver, which in turn stores a reference to its buffer, and in returning the display from our `new` function, the driver and buffer gets dropped, causing LVGL to hold a dangling reference.
There are two ways around this - either, we can `Box` the display driver and store that in the `Gui` struct too, so our reference is always pointing to the same place on the heap, or we can store the driver and buffer directly and create a self referential struct.
The former is much easier to implement in Rust, but has the very small disadvantage of an extra heap allocation (and the same again when we add support for input devices).
The heap allocation shouldn't be an issue in this situation since all the memory will get cleaned up once the kernel takes over, so I'll explore that route.

# Boxing it up

This seems like it should be simple enough - stick `driver` and `buffer` into a `Box`, store the `Box` inside `Gui` and all should work.
Unfortunately it's not that simple.
Something I skipped over in the previous code snippet is that we need to name some lifetimes for our driver and display.

```rust
pub struct Gui {
	display: Display</* lifetime of driver */, /* lifetime of flush function */, /* lifetime of buffer */>,
	driver: Box<Driver</* lifetime of flush function */, /* lifetime of buffer */>>,
	buffer: Box<DrawBuffer>;
}
```

Let's start with the flush function. I say we need to know the lifetime of the flush function, but what does this even mean?
We passed in a closure by moving it, not by reference, so why should there be any lifetime needed?
The truth is, the closure contains some hidden references.
Looking back at our closure, we see that it captures `gop`, which itself is a reference.

```rust
/* The implicit 'a up here... */
pub fn new<'a>(gop: &'a mut GraphicsOutput) -> Self {
	...
	/* is the same as the unnameable lifetime of our closure here */
	let _: {closure}<'a> = |update| Self::flush_display(gop, update);
}
```

So if it's the same as the lifetime of our <code>gop</code> argument, then we can just add it as a generic lifetime on our <code>Gui</code> struct, and require that lifetime to be the same as the one we pass into <code>new</code>. Due to lifetime elision, <code>rustc</code> automatically infers that if one argument and the return type both contain a single lifetime, then they must be the same, leaving our code as

```rust
pub struct Gui<'gop> {
	display: Display</* todo */, 'gop, /* todo */>,
	driver: Box<Driver<'gop, /* todo */>>,
	buffer: Box<DrawBuffer< } impl Gui>'_< { pub fn new(gop: &mut GraphicsOutput) -< Gui>'_< { ... } }
```

So what's the lifetime of our driver? It's valid as long as the <code>Box</code> isn't dropped, and the <code>Box</code> only gets dropped when <code>Gui</code> gets dropped - so it's the lifetime of the object itself? But how do we name this <code>'self</code> lifetime? Unfortunately there isn't a way, and that starts to make sense when you consider drop order. If we take the following code and run it

```rust
struct PrintOnDrop(&'static str);

impl Drop for PrintOnDrop {
	fn drop(&mut self) {
		println!("{}", self.0);
	}
}

struct A {
	one: PrintOnDrop,
	two: PrintOnDrop,
}

fn main() {
	let _ = A { one: PrintOnDrop("one"), two: PrintOnDrop("two") };
}
```

we end up with the output <code>one</code> then <code>two</code>. Swapping the <code>one</code> and <code>two</code> fields round causes them to be dropped the other way round. What this means for us then, is that even if we could use our pretend <code>'self</code> lifetime here, depending on the order that we declare our struct fields in, the driver may get dropped <strong>before</strong> our display, leaving a (short-lived) dangling reference. The only way to get around this is with a bit of lying and <code>unsafe</code>, using a technique I saw in (Self-referential types for fun and profit)[https://morestina.net/blog/1868/self-referential-types-for-fun-and-profit]. First, we ensure that our <code>Gui</code> struct is declared to drop each reference before the field it references - first the display, then the driver, and finally the buffer. However, we can't use the builtin <code>Box</code> type. The builtin <code>Box</code> type is declared as

```rust
pub struct Box<T: ?Sized, A: Allocator = Global>(Unique<T>, A);
```

Importantly, it uses a <code>Unique</code> rather than a <code>NonNull</code>, which carries the extra requirement that it has unique ownership over its contents. For us, this means that even though the contained object has a stable address on the heap, doing the following is still UB.

```rust
let mut x = Box::new(5);
let mut x_ptr = &mut *x as *mut i32;
let mut y = x; // move the box
x_ptr.write(3);
```

Instead, we need to use the <code>AliasableBox</code> type from (aliasable)[https://crates.io/crates/aliasable].

```rust
pub fn new(gop: &mut GraphicsOutput) -> Gui<'_> {
	let (width, height) = gop.current_mode_info().resolution();
	lvgl2::init();
	
	let mut buffer = AliasableBox::from_unique(Box::new(DrawBuffer::new(8000)));
	let mut driver = AliasableBox::from_unique(Box::new(Driver::new(
		&mut buffer,
		width,
		height,
		|update| Self::flush_display(gop, update),
	)));
	
	let display = Display::new(&mut driver);
	Gui { display, driver, buffer }
}
```

Unfortunately we're still not done, and <strong>still</strong> get borrow checker errors.

<code style="white-space: pre;">
<span style="color: #fc5e53; font-weight: bold;">error[E0597]</span>: `buffer` does not live long enough
<span style="color: #6eb5fa; font-weight: bold;"> --></span> bootloader/src/framebuffer/mod.rs:10:4
<span style="color: #6eb5fa; font-weight: bold;">   |</span>
<span style="color: #6eb5fa; font-weight: bold;"> 5 |</span>   let mut buffer = AliasableBox::from_unique(Box::new(DrawBuffer::new(8000)));
<span style="color: #6eb5fa; font-weight: bold;">   |       ---------- binding `buffer` declared here</span>
<span style="color: #6eb5fa; font-weight: bold;"> 6 | /</span> let mut driver = AliasableBox::from_unique(Box::new(Driver::new(
<span style="color: #6eb5fa; font-weight: bold;"> 7 | |</span>     &mut buffer,
<span style="color: #6eb5fa; font-weight: bold;">   | |</span> <span style="color: #fc5e53; font-weight: bold;">    ^^^^^^^^^^^ borrowed value does not live long enough</span>
<span style="color: #6eb5fa; font-weight: bold;"> 8 | |</span>     width,
<span style="color: #6eb5fa; font-weight: bold;"> 9 | |</span>     height,
<span style="color: #6eb5fa; font-weight: bold;">10 | |</span>     |update| Self::flush_display(gop, update),
<span style="color: #6eb5fa; font-weight: bold;">11 | |</span> )));
<span style="color: #6eb5fa; font-weight: bold;">   | |_- argument requires that `buffer` is borrowed for `'static`</span>
<span style="color: #6eb5fa; font-weight: bold;">...</span>
<span style="color: #6eb5fa; font-weight: bold;">15 |</span> }
<span style="color: #6eb5fa; font-weight: bold;">   | - `buffer` dropped here while still borrowed</span><br>
<span style="color: #fc5e53; font-weight: bold;">error[E0505]</span>: cannot move out of `buffer` because it is borrowed
<span style="color: #6eb5fa; font-weight: bold;"> --></span> bootloader/src/framebuffer/mod.rs:23:2
<span style="color: #6eb5fa; font-weight: bold;">   |</span>
<span style="color: #6eb5fa; font-weight: bold;"> 5 |</span>   let mut buffer = AliasableBox::from_unique(Box::new(DrawBuffer::new(8000)));
<span style="color: #6eb5fa; font-weight: bold;">   |       ---------- binding `buffer` declared here</span>
<span style="color: #6eb5fa; font-weight: bold;"> 6 | /</span> let mut driver = AliasableBox::from_unique(Box::new(Driver::new(
<span style="color: #6eb5fa; font-weight: bold;"> 7 | |</span> &mut buffer,
<span style="color: #6eb5fa; font-weight: bold;">   | | ----------- borrow of `buffer` occurs here</span>
<span style="color: #6eb5fa; font-weight: bold;"> 8 | |</span>     width,
<span style="color: #6eb5fa; font-weight: bold;"> 9 | |</span>     height,
<span style="color: #6eb5fa; font-weight: bold;">10 | |</span>     |update| Self::flush_display(gop, update),
<span style="color: #6eb5fa; font-weight: bold;">11 | |</span> )));
<span style="color: #6eb5fa; font-weight: bold;">   | |_- argument requires that `buffer` is borrowed for `'static`</span>
<span style="color: #6eb5fa; font-weight: bold;">...</span>
<span style="color: #6eb5fa; font-weight: bold;">14 |</span> Gui { display, driver, buffer }
<span style="color: #6eb5fa; font-weight: bold;">   |</span> <span style="color: #fc5e53; font-weight: bold;">                       ^^^^^^ move out of `buffer` occurs here</span></code>

The issue here is that we told the borrow checker to expect <code>'static</code> references, and then gave it references scoped to the current function, and even worse, moved the object the reference came from, triggering both an error about not living long enough <strong>and</strong> one about dropping the owner while it's borrowed. The fix for this is simple, if slightly scary - we need to adjust the lifetime of <code>&mut buffer</code> and <code>&mut driver</code> with a <code>transmute</code>. In this situation, we know this is safe since we manually made sure that the references are never dangling.

```rust
let mut driver = AliasableBox::from_unique(
	Box::new(
		Driver::new(
			unsafe { transmute(&mut *buffer) },
			width,
			height,
			|update| Self::flush_display(gop, update)
		)
	)
);
let display = Display::new(unsafe { transmute(&mut *driver) });
```

However, there is one <strong>extremely important</strong> difference on top of the <code>transmute</code> - I've replaced <code>&mut buffer</code> with <code>&mut *buffer</code> (and same for <code>driver</code>). Previously we were fine without the explicit dereference, since the compiler automatically inserted one to make the expected type line up. <code>transmute</code> has no care about the types here, and will happily turn a <code>&Box>T<</code> into a <code>&T</code> which is really not what we want (and ends up with an invalid opcode (somehow)). The addition of the explicit dereference forces the <code>transmute</code> input to be a <code>&T</code>.</p> <!-- /wp:paragraph --> <!-- wp:paragraph --> <p>And if you thought this was complicated, now you see why I didn't explore the option of not even using <code>Box</code>es.</p>

---

With all that out of the way, now we can add a basic UI and an event loop. I won't go into any detail about the UI, since that becomes pretty specific to what you want to achieve. It works fairly similarly to the [LVGL C API](https://docs.lvgl.io), just object oriented and using RAII rather than manually calling delete functions, something like this

```rust
let mut screen = ui.display.active_screen();
let mut style = screen.inline_style(Part::Main, State::DEFAULT);
style.set_bg_color(Color::from_rgb(0x33, 0x33, 0x33));
style.set_text_color(Color::from_rgb(0xee, 0xee, 0xee));
style.set_bg_opa(Opacity::OPA_100);
let mut flex_box = Object::new(Some(screen));
```

However, LVGL won't actually render anything without an event loop. We need to call LVGL's <code>timer_handler</code> function at short intervals, and it will redraw the display, poll input devices and run animations. So it can properly run animations and other time based functions, it also needs to know how frequently it's being called, which we do with the <code>tick_increment</code> function. I tried messing around with the RTC but, at least in my testing with qemu, it doesn't support sub-second precision, making it a bit useless for animations. Instead, I've gone with the UEFI timer system, which allows us to set a callback to run at a periodic interval. To do this, first we have to write our callback method. Since it's called by the UEFI subsystems, it needs to use the UEFI calling convention, so we add <code>extern "efiapi"</code> before the function definition. It also takes two parameters so we can figure out why it was called, but since we only use this callback for the timer we can ignore them.

```rust
extern "efiapi" fn timer_callback(_: Event, _: Option<NonNull<core::ffi::c_void>>) {
	lvgl2::timer_handler();
	lvgl2::tick_increment(Duration::from_millis(30));
}
```

Then, we can ask the UEFI firmware to actually call the callback. First, we create an 'event' using the <code>BootServices::create_event</code> function. We want a timer event, and we want it to call our callback when it's signalled, so we set the type to be <code>TIMER | NOTIFY_SIGNAL</code>, as well as passing our callback as the notify function. It also requires a priority level to run the callback at - UEFI recommends keeping this as low as possible, so we'll use the <code>CALLBACK</code> level, which is the minimum level for a callback to run at. <code>create_event</code> is unsafe, as after we exit boot services, the callback may still try to access them. Since we never exit boot services yet, this is fine for us, but we need to remember to look at it again later, since the GOP that we use in LVGL is only available before exiting boot services. Now we need to tell UEFI when to signal our event. We want it to be called by the timer periodically at (in my case) 30ms intervals - note however that UEFI expects the period to be given in 100ns intervals (centimilliseconds?), so this is actually a value of 300.

```rust
let timer_event = unsafe { boot_services.create_event(
	EventType::TIMER | EventType::NOTIFY_SIGNAL,
	Tpl::CALLBACK,
	Some(timer_callback),
	None
) }.unwrap();

boot_services.set_timer(&timer_event, TimerTrigger::Periodic(300)).unwrap();
```

We also need to prevent the <code>main</code> function from returning, as that will cause our bootloader to close, by just adding an empty loop after setting our timer. Running the code now should bring up your LVGL UI on screen. But if you leave it running for around 5 minutes, it seems to suddenly shut itself off. The problem here is that to prevent an unresponsive bootloader from locking up the system, the firmware sets up what's called a watchdog timer - a timer implemented in hardware that automatically shuts down a system if it doesn't reset the timer fast enough. Useful for safety critical systems, but less useful if we want to wait an indefinite amount of time for user input. So we can just turn it off!

```rust
boot_services.set_watchdog_timer(0, 0x10000, None).unwrap();
```

The special timeout value of 0 is reserved to mean disabling the watchdog timer. Like with our timer callback, the watchdog also allows us to specify some context that gets logged when the system is reset. In our case, we don't need any since we're disabling the timer, so we set the data to <code>None</code> and the code to <code>0x10000</code> (codes below that are reserved for use by the firmware, so even though we're disabling the watchdog, we can't use them).

---

Hopefully now you've been able to set a UI in your bootloader with LVGL. In the next post I'll explore getting user input, and once they click a button, actually boot the kernel.

[^1]: I actually had another build issue after fixing this - when building for non-host targets <code>clang</code> seems to append the target name to the system include path, so for me it was trying to look in <code>/Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/x86_64-w64-mingw32/usr/include</code>. There are two ways to fix this - either add back the original include path and use the (wrong) system headers. This may or may not work properly depending on the system and C stdlib functions that LVGL expects. Some of the C stdlib is implemented by Rust (such as <code>strlen</code> and <code>memcpy</code>) but other parts may not be. The correct way is to get hold of the UEFI C stdlib, and link with that instead.