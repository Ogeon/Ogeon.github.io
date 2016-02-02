---
layout: post
title: "Palette 0.2.0: A Different Shade"
---

It has now been nearly three weeks since version 0.1.0 of
[Palette][palette_git], a linear color calculation and conversion library for
Rust, was released, but it has already gone through some big and important
changes, so I thought it would be a good idea to describe them in a bit more
detail. Let's take a look at what's new.

## Slicing Gradients

Palette provides a `Gradient` type for linear color interpolation. The basic
idea is that you can define a series of control points (colors mapped to
numbers), and the `Gradient` will mix them to give you any color between them.
The colors beyond the endpoints of the series will be the same as at the
nearest endpoint.

The new slicing feature lets you create an other gradient, which is based on
the original one, but with a different domain (or range). Everything outside
this new domain will have the same color as at the nearest of its endpoints,
so it's the same behavior as before, but with new endpoints. Let's take a look
at an example:

```rust
//Create a gradient from red to blue. The default domain is 0.0 to 1.0.
let g1 = Gradient::new(vec![Rgb::new(1.0, 0.0, 0.0), Rgb::new(0.0, 0.0, 1.0)]);

//Slice it to only include the first half.
let g2 = g1.slice(..0.5);

assert_eq!(g1.get(0.0), g2.get(0.0)); //Ok
assert_eq!(g1.get(0.3), g2.get(0.3)); //Ok
assert_eq!(g1.get(0.5), g2.get(0.5)); //Ok
assert_eq!(g1.get(0.7), g2.get(0.7)); //Error!
```

The reason for the error is that `0.7` is outside the domain of `g2`, so it
gives `RGB(0.5, 0.0, 0.5)` (the color at `0.5`) instead of `RGB(0.3, 0.0, 0.7)`.

See [the documentation][gradient_docs] for more info about `Gradient`.

## Color Arithmetics

One of the two main purposes of Palette is doing maths with colors and that
has now been made a lot simpler. The basic arithmetic operations (`+`, `-`,
`*` and `/`) has been implemented for each color space, and allows both
another color and a plain number as the right hand side.

```rust
let a = Rgb::new(1.0, 0.5, 0.3);
let b = Rgb::new(0.1, 0.5, 0.2);

//It's now possible to do this:
let c = a * b;

//...instead of this:
let c = Rgb::new(a.red * b.red, a.green * b.green, a.blue * b.blue);

//...and this:
let c = a * 0.5;

//...instead of this:
let c = Rgb::new(a.red * 0.5, a.green * 0.5, a.blue * 0.5);
```

The only exceptions are the hue based colors (HSV, HSL, etc.), since
multiplication and division of the hue isn't well defined. They do still
implement addition and subtraction, so they are not completely left outside.

## Working With Any Float

The colors were initially represented by `f32` components, which may be too
limited for some high precision applications, so every single color type was
rewritten to be based on any `T: num::Float`. The difference isn't directly
apparent, since the default is still `f32`, but it's now possible to use
`Rgb<f64>` or even `Rgb<MyIncreatibelFloatType>`.

The operation traits, like `Mix` and `Shade`, has been given an associated
type, called `Scalar`. This corresponds to the type parameter `T` in the color
spaces and allows some simplification when they are used together, like in
`Gradient<C: Mix + Clone>`.

## Separating Transparency From Color

The second big change was the separation of the transparency (alpha) component
from the colors. The `alpha` was initially a mandatory part of each color, but
this came with increased memory usage if many colors are to be stored, so some
kind of decoupling was necessary.

One idea was to make duplicates of each color type, where one has the `alpha`
component and the other doesn't. This is simple and can easily be automated.
The other idea was to make more use of the type system and create a
transparent wrapper type which would carry the `alpha` component. This ended
up being the final solution and comes with some interesting possibilities.

The new system introduces an `Alpha<C, T: Float>` type, which is defined as

```rust
pub struct Alpha<C, T: Float> {
    color: C,
    alpha: T,
}
```

This type implements all of the operation traits, as well as `Deref` and
`DerefMut` to also expose the content of `color`. This allows operations on
`color` + `alpha` as a whole, as well as isolated operations on just `color`
without any conversion. Take a look at this example from the documentation:

```rust
use palette::{Rgb, Rgba};

let mut c1 = Rgba::new(1.0, 0.5, 0.5, 0.8);
let c2 = Rgb::new(0.5, 1.0, 1.0);

c1.color = c1.color * c2; //Leave the alpha as it is
c1.blue += 0.2; //The color components can easily be accessed
c1 = c1 * 0.5; //Scale both the color and the alpha
```

The `Rgba` type is an alias for `Alpha<Rgb<T>, T>` and it implements the
necessary functions for it to behave in the same way as `Rgb`. There are also
aliases for the other color spaces.

You can read a bit more about `Alpha` in [the documentation][alpha_docs].

## Separation of Pixel Encodings

An other important change is the separation of sRGB and gamma correction
related things from the `Rgb` type. The distinction between linear and non-
linear colors is now encoded into the type system as the `Srgb` and `GammaRgb`
types. They are meant as transition types when converting from a linear color
to some kind of pixel representation, and back. This change has also resulted
in a `pixel` module, where all the pixel format related types can be found.

See [the documentation][pixel_docs] for more.

## New Constructor Names

Last, but not least, is a short PSA regarding the new constructor names. The
separation of transparency, and the new RGB types made the old constructor
naming convention (like `Rgb::rgb(...)`) unnecessary. Each type (`Rgb`,
`Rgba`, `Srgb`, etc.) can now be constructed with a `new` function, and
sometimes with an additional `new_u8` function. The `Color` type is mostly as
before, though, with the exception of the transparency variants.

## What's Next?

The next phase will focus on adding some new features. There are already [some
to-do issues][issues] that has been scheduled for version 0.2.1, including
named colors, blending and the addition of xyY support. A further goal is to
implement variable white points and chromatic adaption.

Palette can be found [on GitHub][palette_git], where contributions are most
welcome, and on [crates.io][crates].

Thank you for reading this, and have fun making colorful creations!



[palette_git]: https://github.com/Ogeon/palette
[gradient_docs]: https://ogeon.github.io/docs/palette/master/palette/gradient/struct.Gradient.html
[alpha_docs]: https://ogeon.github.io/docs/palette/master/palette/struct.Alpha.html
[pixel_docs]: https://ogeon.github.io/docs/palette/master/palette/pixel/index.html
[issues]: https://github.com/Ogeon/palette/issues
[crates]: https://crates.io/crates/palette
