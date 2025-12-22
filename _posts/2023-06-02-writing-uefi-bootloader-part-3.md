---
layout: post
title: "Writing a UEFI bootloader - part 3"
---
With filesystem access in place, we can start working on loading images from the disk, and putting together a basic UI. Once that's in place, we should be able to load a kernel image from disk, and boot into it.

# The Targa image format

Having messed around with BMP images a few times before (including a very strange competition of trying to write a block colour image to disk as fast as possible), I was initially going to start this by writing (or finding a crate for) a BMP parser. However, while looking into font rendering, I came across this [OSDev wiki page](https://wiki.osdev.org/Loading_Icons), which specifically states not to use BMP. I've never had the issues it mentions, but then most of my work in the past has been writing BMP files for other software to read, rather than reading them myself. On top of that, the Targa format also looked slightly easier to write quickly, so I ended up going with it.

Initially I had a not-very-thorough look on crates.io, before coming to the conclusion that I could write something 'better' (read: more bugs and fewer features). I did come across [embedded-graphics](https://crates.io/crates/embedded-graphics) which it seems like has Targa support, as well as other useful rendering utilities. Unfortunately, a brief look at it made it seem like framebuffer pixel format is done at compile time with generics, which is less helpful when dealing with arbitrary hardware in the bootloader.

The Targa parser I've written so far is extremely simple and feature-lacking, but tries to be efficient and future-proof. It currently only supports 24 and 32 bit colour, decompressed images, however I plan to try supporting run length encoded images in future. To reduce memory allocations within the library, I may have slightly abused <code>alloc::borrow::Cow</code>. This is the output of the parse function:

```rust
pub struct Image<'a> {
	pixel_data: Cow<'a, [u8]>,
	width: usize,
	...
}
```

The idea is that if the image needs no decompression, the <code>pixel_data</code> can just be a reference directly into the image in memory, removing any need for allocations. In this case it can return the <code>Borrowed</code> state of <code>Cow</code>. If the image needs decompression, then the parsing function can allocate a `Vec<u8>` for the decompressed data, and return the <code>Owned</code> form of <code>Cow</code>.

# A safer framebuffer

A couple of posts ago, I set up some very basic framebuffer rendering (if you can even call a red screen rendering) using some unsafe raw pointers. Now that I know it works, I'll set up a safe wrapper around it, with some utilities to draw Targa images.

We can start by creating a <code>Framebuffer</code> struct to contain the resolution and framebuffer pointer, then make a <code>new</code> function to find a sensible resolution if possible, and return the struct:

```rust
pub fn new_from_gop(gop: &mut GraphicsOutput) -> Self {
	let optimal_resolutions = gop.modes()
		.filter(|mode| mode.info().resolution() == (1920, 1080)
			|| mode.info().resolution() == (1280, 720)
			|| mode.info().resolution() == (640, 480)
		);
	let optimal_resolution = optimal_resolutions.reduce(|acc, mode| {
		if mode.info().resolution().0 > acc.info().resolution().0 { mode }
		else { acc }
	});
	
	let actual_resolution = if let Some(resolution) = optimal_resolution && gop.set_mode(&resolution).is_ok() {
		resolution.info().resolution()
	} else { gop.current_mode_info().resolution() };
	
	Self { width: actual_resolution.0, height: actual_resolution.1 } }
```

We can then add a stride field, as well as a pointer to the framebuffer and a double-buffer. To ensure the underlying framebuffer lives long enough, I've added a phantom data lifetime, which will be the same as the lifetime of the GOP handle the framebuffer is built on. (I had a look inside the uefi crate and it seems this is the lifetime it uses internally for its own Framebuffer object, but I think it might be possible to break this by changing the resolution to result in a different framebuffer address without causing any lifetime problems.) This results in the final struct looking like this[^1]:

```rust
pub struct Framebuffer<'a> {
	width: usize,
	height: usize,
	stride: usize,
	double_buffer: Box<[u8]>,
	actual_buffer: *mut u8,
	_phantom: PhantomData<&'a mut u8>
}
```

Then a <code>flush()</code> function can be added, and the code from earlier rewritten to now write to the double buffer, rather than writing to VRAM directly. And, somehow or other, it still works!

<!--<figure class="wp-block-image aligncenter size-medium is-resized"><img src="https://eliyahu.co.uk/wp-content/uploads/2023/05/image-3-300x180.png" alt="A screenshot of a qemu window, with a red background and a green checkmark in the upper left corner." class="wp-image-105" width="512" height="308"/></figure>-->

# A proper way to draw

Currently, writing to the framebuffer still needs some raw pointer access to the double buffer, which is suboptimal. To remedy this, we can add some primitive drawing functions. Taking notes from the embedded-graphics crate, I'll make a <code>Drawable</code> trait which can then be implemented for each type of object.

The easiest place to start is with a rectangle. To start, I've made a struct that holds an origin and size, as well as the colour to fill it with. I've also made a <code>Drawable</code> trait which looks like this:

```rust
pub trait Drawable {
	type Output;
	fn draw(&self, framebuffer: &mut Framebuffer) -> Result<Self::Output, OutOfBoundsError>;
}
```

To draw the rectangle, we can just iterate over all the pixels within the rectangle, and set each to the specified colour. There are a few different ways to approach the individual pixel drawing - I went with the option of adding a second <code>Drawable</code> type called <code>Pixel</code> and putting the unsafe framebuffer access within it's <code>draw()</code> function (although I'm now realising maybe it would be better to have the <code>Pixel</code> and <code>Framebuffer</code> types less tightly coupled). Once we have a drawable rectangle, the initial code to make the framebuffer red can just be replaced with a call to <code>Rectangle::draw()</code> - no unsafety in sight.

To draw images, we can follow a similar process - iterate over all the pixels within the image data, and call a draw pixel function for each one. To do this, we can make an iterator over a Targa image which returns each pixel and its position. We also need some way to provide the offset to draw an image at, so I opted for making a wrapper type that contains both the Targa image and the offset to start drawing at. In trying to make it as generic as possible, I ended up with this lovely where clause.

```rust
pub fn new<P>(origin: (usize, usize), image: T) -> Self where for<'a> &'a T: IntoIterator<Item = P>, Pixel: From<P> { ... }
```

The idea is that you have some image type <code>T</code>, on which you can call <code>into_iter(&self)</code>. (I've made <code>IntoIterator</code> a bound on <code>&T</code> rather than <code>T</code> since it's probably better here for the iterator conversion to be non-consuming - that way one image can be drawn many times over and over. The `for<'a>` is a [higher-rank trait bound](https://doc.rust-lang.org/nomicon/hrtb.html) - we need it here since there is no known lifetime that we're calling <code>into_iter()</code> with until we actually call it, and it's a valid call for any lifetime.) The iterator it provides then returns some type which is convertible to a <code>Pixel</code>, to convert it from the image's own pixel format into the correct format to put into the framebuffer. With that set up, the actual <code>draw()</code> function is fairly trivial:

```rust
self.image.into_iter()
	.try_for_each(|px| {
		let mut px = Pixel::from(px);
		px.pos.x += self.origin.x;
		px.pos.y += self.origin.y;
		framebuffer.draw(px)
	})
```

We iterate over the pixels, adding an offset to each one so it draws at the given origin, and then draw the pixel. If we combine this all together with the file access from the previous post, it should now be possible to make a very simple, non-interactive (for now) GUI. You could even try adding more primitive objects, or add more complexity to the existing rectangle object, like transparency.

---

With a basic GUI together, we can now start working on keyboard input and kernel loading (in one order or another).

[^1]: I know technically that I should also be worrying about colour format but currently everything is just inside qemu which seems to use BGR, so I'm leaving this for now