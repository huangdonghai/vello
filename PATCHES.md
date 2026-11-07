# Local patches on this branch

`bitmap-glyphs` is upstream `v0.11.0` plus the fixes Cuotuo's browser needs. It
is consumed as a git submodule at `deps/vello` and through `[patch.crates-io]`
pointing at `deps/vello/research/vello_research`, which is where the published
`vello` 0.11.0 crate lives in this tree.

## `draw_glyphs` and fonts with embedded bitmap strikes

`research/vello_research/src/scene.rs`, `DrawGlyphs::try_draw_colr`.

`glyph_for_size` returns the *nearest* bitmap strike whatever size is asked for,
so a font carrying a 20px strike rendered every size from it — and Vello
prefers a strike to the outline whenever the font has one. Two problems
followed, both silent:

* The decoded sample of a monochrome mask is 0 or 1 (the bit depth's own range),
  but it was written straight into an 8-bit alpha channel, so the glyph was
  composited at alpha 1/255 — invisible, and white, because the colour channels
  are `u8::MAX`. MS Gothic and SimSun ship 1-bpp strikes and disappeared
  entirely.
* Row-padded ("unpacked") masks were dropped with a warning, which took SimSun
  with them even where the strike did cover the requested size.

A monochrome strike is now used only at the size it was designed for — anything
else falls through to the outline, as browsers do — its samples are scaled to
0..=255, and unpacked rows are decoded. Colour bitmaps (`Bgra`/`Png`: emoji,
`sbix`, `CBDT`) are untouched.

## Which side of the y-flip a glyph transform is composed on

`research/vello_encoding/src/resolve.rs`, and `DrawGlyphs::try_draw_colr` in
`research/vello_research/src/scene.rs`.

`glyph_transform` is stated in layout space — the space a run's glyph positions
are already in. The GPU encoder composed it on the font-space side of the flip
that carries an outline from y-up to y-down, which conjugates it: a quarter turn
clockwise came out counter-clockwise, and a synthetic oblique leaned the other
way. `vello_cpu` (through `glifo`) applies it after the flip, so the two
rasterizers of the same page disagreed on which way vertical text reads:
`text-orientation: sideways` ran down the column on the CPU and up it on the
GPU, and the Latin of a `mixed` column was turned as the author's browser would
not turn it.

The same mistake sat in the two branches of `try_draw_colr`, where turning the
unit square a bitmap or a COLR glyph is drawn from moves the glyph by its own
place rather than turning it. Each now composes the turn between the glyph's own
placement and the offsets inside it, which is measurable for a colour emoji: a
`😀` at 20px set sideways in a column came out 229 dark pixels a dozen pixels
left of where the CPU puts it, and 408 where the CPU puts 409. A struck monochrome
bitmap could not be checked this way, because those paint nothing at all on the
GPU even unturned — a separate fault, not this one.

Measured against Chrome for the outline path: `F`, `p` and `中` at 96px in
`vertical-rl` agree on both backends for `mixed`, `sideways` and `upright`, and
`samples/.../Pages/vertical-mixed.htm` comes to the same picture on the GPU as on
the CPU.
