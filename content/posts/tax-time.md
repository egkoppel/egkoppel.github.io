---
draft: true
title: Tax time (sadly)
date: 2026-04-10
---
With the April 15th deadline for my US tax payment in less than a week, I've realised now is probably the least worst time to actually get on with figuring out if I actually owe anything.
Through a big mess of dealing with accidental historic non-compliance, I realised around three years ago that I have to deal with the horror of IRS form 8621 and section 1291 funds.
Having read the advice of
> go seek a professional.
{cite="Reddit"}

and
> The estimated burden for all other taxpayers who file this form is shown below.
> |   |   |
> |---|---|
> |Recordkeeping|16 hr., 58 min.|
> |Learning about the law or the form|11 hr., 24 min.|
> |Preparing and sending the form to the IRS|20 hr., 34 min.|
{cite="IRS"}

I felt extremely encouraged in my choice to deal with form 8621 entirely myself, having zero prior knowledge about filing tax returns other than "it takes too much effort"!
Having decided that form8621.com was too expensive, and 8621calculator.com not having existed (as best as I can find based on WHOIS data) back then, I plowed into the depths of the approximately ~14.5k words of "help" the IRS gives on its instruction page for the form.

# Time for some software!

Doing a basic time analysis using the chart from xkcd 1205 shows that if I shave 6 hours off the filing time per year (which seems like a reasonable assumption given that it now takes me a couple hours from start to finish across multiple copies of the form, compared to the IRS estimates of **37 hours per form**), it's well worth wasting a day of my life turning word soup into spaghetti software.

![xkcd 1205 - Is It Worth The Time?](https://imgs.xkcd.com/comics/is_it_worth_the_time.png)

My two points of reference were the IRS instructions, and the example output from form8621.com.
And so with those ready to read/swear at/cry at, I sat down and started typing.

# This is actually hard

My initial plan was to try and make an actually usable piece of software.
Have some vaguely nice way to enter data (apparently TOML because I'm too used to Rust and writing a TOML file is apparently considered "nice" by me), process the entered data and run calculations, and then output an overly long PDF to print out and post to the US.
Needless to say that got scaled back very quickly.

I initially planned to build this in Python, with the thinking that it's basically equivalent to a bunch of data analysis.
Maybe I'm too Rust-pilled at this point, but within a few minutes of trying to put something together, I was already missing my type system and the ease of just slapping `#[derive(Deserialize)]` on everything I needed to get from the user.
So I rewrote it in Rust!

Many tedious hours of writing code later, and cross-checking output data with the example from form8621.com, I finally had a working, extremely cursed dataflow pipeline.
I would go through all my account statements for PFICs, convert the data into a long TOML file, and run that through 1500 lines of Rust.
This would then use some extremely cursed format strings to generate a LaTeX file of calculation workings, which I would then manually edit to number and title them properly, render to a PDF, and then finally copy the final values from the working PDF over to form 8621.
And then figure out all the rest of the return manually...
# Making it pretty

With some small changes needed for the 2025 tax year, I decided the best way to focus on those was to spend my time making a nice looking frontend instead.

>[!WARNING]
>TODO

And with all that out of the way, it's probably time to actually do my taxes