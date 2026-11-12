---
name: drag-and-drop
description: >
  Best practices for implementing HTML5 drag-and-drop reordering with slide
  animations. Use when adding, debugging, or extending drag-and-drop in any
  project. Covers the placeholder pattern, FLIP animation, common pitfalls,
  and errors to avoid.
---

# Drag-and-Drop with Slide Animation

Covers the **placeholder pattern** and **FLIP animation** for smooth reorder feedback in a draggable grid or list.

---

## Core Architecture

```
dragstart  → hide real cell, insert placeholder clone
dragover   → move placeholder to new position (triggers FLIP animation)
drop       → persist new order
dragend    → replace placeholder with real cell, clean up
```

---

## The Placeholder Pattern

Instead of moving the dragged element itself, clone it as a `.placeholder` and hide the original. This keeps the drag image intact and gives a clear "gap" effect.

```js
let placeholder = null;

item.addEventListener('dragstart', e => {
  e.dataTransfer.effectAllowed = 'move';
  e.dataTransfer.setData('text/plain', item.dataset.id); // required in Firefox
  item.classList.add('dragging');

  placeholder = item.cloneNode(true);
  placeholder.classList.remove('dragging');
  placeholder.classList.add('placeholder');
  item.after(placeholder);                        // insert gap right away

  setTimeout(() => { item.style.display = 'none'; }, 0); // defer hide so drag image renders
});

item.addEventListener('dragend', () => {
  if (placeholder?.parentNode) placeholder.replaceWith(item); // restore real item
  item.style.display = '';
  item.classList.remove('dragging');
  placeholder = null;
  commitOrder(container);
});
```

---

## Locating the Drop Position (`getDragAfterElement`)

Finds which element the dragged item should be inserted **before**, based on cursor X position (swap for `clientY` on vertical lists).

```js
function getDragAfterElement(container, x) {
  // ⚠️ MUST exclude both .dragging AND .placeholder
  const items = [...container.querySelectorAll('.item:not(.dragging):not(.placeholder)')];
  return items.reduce((closest, child) => {
    const box = child.getBoundingClientRect();
    const offset = x - box.left - box.width / 2;
    if (offset < 0 && offset > closest.offset) {
      return { offset, element: child };
    }
    return closest;
  }, { offset: Number.NEGATIVE_INFINITY, element: null }).element;
}
```

> For **vertical lists** replace `x` / `clientX` / `box.left` / `box.width` with `y` / `clientY` / `box.top` / `box.height`.

---

## FLIP Slide Animation (`animateSlide`)

Wraps a DOM mutation so surrounding elements animate into their new positions.

```js
function animateSlide(container, mutate) {
  const items = [...container.querySelectorAll('.item')];
  const first = new Map(items.map(el => [el, el.getBoundingClientRect()])); // snapshot BEFORE
  mutate();
  items.forEach(el => {
    const last = el.getBoundingClientRect();
    const prev = first.get(el);
    if (!prev) return;
    const dx = prev.left - last.left;
    const dy = prev.top  - last.top;
    if (!dx && !dy) return;
    el.style.transition = 'none';
    el.style.transform  = `translate(${dx}px, ${dy}px)`;
    requestAnimationFrame(() => requestAnimationFrame(() => { // double-rAF to flush paint
      el.style.transition = 'transform 0.15s ease-out';
      el.style.transform  = '';
      el.addEventListener('transitionend', function clear() {
        el.style.transition = '';
        el.removeEventListener('transitionend', clear);
      });
    }));
  });
}
```

Usage inside `dragover`:

```js
container.addEventListener('dragover', e => {
  e.preventDefault();
  e.dataTransfer.dropEffect = 'move';
  if (!placeholder) return;
  const after = getDragAfterElement(container, e.clientX);
  // skip if placeholder is already in the right spot
  if (after === placeholder.nextElementSibling) return;
  if (after == null && placeholder === container.lastElementChild) return;
  animateSlide(container, () => {
    if (after == null) container.appendChild(placeholder);
    else container.insertBefore(placeholder, after);
  });
});
```

---

## Persisting Order

```js
// on BOTH drop and dragend — belt-and-suspenders
container.addEventListener('drop', e => {
  e.preventDefault();
  commitOrder(container);        // ⚠️ do not omit — dragend may not fire on all browsers
});

function commitOrder(container) {
  const newOrder = [...container.querySelectorAll('.item')].map(el => el.dataset.id);
  // persist newOrder (localStorage, server call, state update…)
}
```

---

## Errors to Avoid

| ❌ Mistake | ✅ Fix |
|---|---|
| Declaring `items` inside the function then deleting the declaration during a refactor, while keeping `items.reduce(...)` | Always declare the query **inside** `getDragAfterElement` — never rely on an outer variable |
| Excluding only `.dragging` in the selector — but the real item is `display:none` and replaced by `.placeholder` | Exclude **both**: `.item:not(.dragging):not(.placeholder)` |
| Removing `commitOrder()` from the `drop` handler during a refactor | Keep it in **both** `drop` and `dragend`; `dragend` alone is unreliable cross-browser |
| Calling `item.style.display = 'none'` synchronously in `dragstart` | Wrap in `setTimeout(..., 0)` — the browser needs one tick to capture the drag image first |
| Animating the hidden dragged item in `animateSlide` | The dragged item has `display:none` — `getBoundingClientRect()` returns zeros. Guard with `if (!prev) return` |
| Forgetting `e.dataTransfer.setData(...)` in `dragstart` | Required by Firefox; without it, `drop` never fires |

---

## CSS Checklist

```css
.item { position: relative; }       /* needed if items have absolute children */

.item.dragging    { opacity: 0.4; }  /* feedback on the drag image */
.item.placeholder {                  /* the gap left behind */
  opacity: 0.35;
  border-style: dashed;
}
```

---

## References

See `references/flip-animation.md` for a deeper explanation of the FLIP technique.
