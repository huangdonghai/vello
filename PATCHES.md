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
