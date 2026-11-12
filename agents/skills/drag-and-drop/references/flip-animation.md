# FLIP Animation — Deep Reference

**FLIP** = **F**irst → **L**ast → **I**nvert → **P**lay

It is the standard technique for animating layout changes that CSS transitions can't directly handle (e.g. `insertBefore`, `appendChild`).

## Steps

### 1. First — snapshot positions before the mutation

```js
const first = new Map(cells.map(c => [c, c.getBoundingClientRect()]));
```

### 2. Last — apply the DOM mutation

```js
mutate(); // e.g. container.insertBefore(placeholder, after)
```

After `mutate()`, elements are in their new layout positions.

### 3. Invert — push elements back to where they came from

```js
const dx = prev.left - last.left;
const dy = prev.top  - last.top;
cell.style.transition = 'none';          // disable transition during inversion
cell.style.transform  = `translate(${dx}px, ${dy}px)`;
```

The element is now *visually* in its old position even though it's *logically* in its new one.

### 4. Play — release the inversion

```js
requestAnimationFrame(() => requestAnimationFrame(() => {
  cell.style.transition = 'transform 0.15s ease-out';
  cell.style.transform  = '';
}));
```

The double `requestAnimationFrame` ensures the browser has actually painted the inverted frame before releasing, preventing the animation from being skipped.

## Why double `requestAnimationFrame`?

A single `rAF` can run in the same paint cycle as the style change. The second `rAF` guarantees a full repaint has occurred so the browser has something to transition *from*.

## Pitfalls

- **Don't animate elements with `display:none`** — `getBoundingClientRect()` returns a zero rect; the inversion will be nonsensical. Guard: `if (!prev || (!prev.width && !prev.height)) return;`
- **Clear `transition` after the animation ends** — leaving `transition: transform 0.15s` on the element permanently can interfere with other animations. Use `transitionend` to clean up.
- **Don't snapshot inside the mutate callback** — by definition FLIP requires the snapshot to happen *before* the mutation.
- **Don't use FLIP for elements being removed** — the "Last" rect for a removed element doesn't exist. Handle exits separately (e.g. a fade-out before removal).
