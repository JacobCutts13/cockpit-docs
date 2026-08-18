# 10. The design system

A tool you sit in all day is a room. Cockpit's is deliberately warm — an earthy palette,
lamp-lit accent, wooden framing, hand-drawn pixel sprites. The stated north star is *a
cozy workshop*, positioned against the default dark-mode dashboard look.

Two tensions are held on purpose:

- **Atmosphere and signal coexist.** Warmth is the resting state; colour-as-meaning rides
  on top of it. The accent means "act here" and is never used to decorate.
- **Cozy is not spacious.** Density stays high because this is an expert's tool. The
  warmth is earned through palette, sprites and framing — never through whitespace.

## One token source, two themes

Every colour is a CSS custom property defined in exactly one block. A theme is a different
set of values for the same tokens, selected by a data attribute on the root element. Two
rules follow, and they are the whole system:

**Never branch on the theme in component code, and never duplicate a colour.** A component
that reads `--panel` is correct in both themes for free.

**Contrast holds both ways.** A dark and a light theme are not one design with inverted
values; each pairing has to be checked independently.

The stored theme is applied *before* the first render, so a persisted choice never
flashes. Components that cannot read CSS variables — the embedded terminal, which needs
concrete colours — subscribe to a change notification rather than each watching the DOM
attribute themselves.

## Sprites as data

The pixel art is not image files. Each sprite is a grid of characters plus a legend
mapping each character to a **colour token**:

```ts
const LEGEND = {
  K: 'var(--sprite-ink)',
  O: 'var(--sprite-wood)',
  A: 'var(--accent)',
  '.': 'transparent',
};
```

Rendered as inline SVG rectangles with crisp-edge rendering. Three properties fall out of
this that raster assets do not give you:

- **The art recolours with the theme automatically**, because every pixel references a
  token rather than a value.
- **It is crisp at any size**, with no asset pipeline and no resolution variants.
- **There are no licensing questions.** Everything is authored in place rather than
  sourced, which was an explicit goal — the alternative is lifting assets from a game.

The sprite name type is derived from the same unions as the pane statuses and sidebar
modes (chapter 5). Adding a status is a compile error until its sprite exists. The set
cannot drift.

Animation is grouped so only the moving part moves — a flame flickers while the candle
holding it stays put — and all of it stops under a reduced-motion preference.

The whole thing sits behind one component, which is deliberately the swap-in seam: a
richer commissioned sprite sheet can replace the implementation later without touching a
single call site.

## Typography

Four families, all bundled locally rather than fetched. That is a design decision with a
security consequence: no remote font requests means the renderer's content policy stays
locked to same-origin, and there is no layout shift on load.

The rule that keeps the aesthetic from becoming a gimmick: **pixel type is a seasoning.**
It appears in the wordmark and nowhere else. Body and interface text is a warm readable
sans. Monospace is *semantic* — it means the text is a machine identifier, not that it
looked nice there.
