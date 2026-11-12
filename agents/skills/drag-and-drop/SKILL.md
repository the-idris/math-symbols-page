---
name: drag-and-drop
description: >
  Best practices for implementing HTML5 drag-and-drop reordering with slide
  animations. Use when adding, debugging, or extending drag-and-drop in this
  project. Covers the placeholder pattern, FLIP animation, common pitfalls,
  and the exact bugs that broke the History section.
---

# Drag-and-Drop with Slide Animation

Covers the **placeholder pattern** (used in this project's History section) and **FLIP animation** for smooth reorder feedback.

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
cell.addEventListener('dragstart', e => {
  e.dataTransfer.effectAllowed = 'move';
  e.dataTransfer.setData('text/plain', key);     // required in some browsers
  cell.classList.add('dragging');

  placeholder = cell.cloneNode(true);
  placeholder.classList.remove('dragging');
  placeholder.classList.add('placeholder');
  cell.after(placeholder);                        // insert gap right away

  setTimeout(() => { cell.style.display = 'none'; }, 0); // defer hide so drag image renders
});

cell.addEventListener('dragend', () => {
  if (placeholder?.parentNode) placeholder.replaceWith(cell); // restore real cell
  cell.style.display = '';
  cell.classList.remove('dragging');
  placeholder = null;
  commitOrder(container);
});
```

---

## Locating the Drop Position (`getDragAfterElement`)

Finds which element the dragged item should be inserted **before**, based on cursor X position.

```js
function getDragAfterElement(container, x) {
  // ⚠️ MUST exclude both .dragging AND .placeholder
  const cells = [...container.querySelectorAll('.cell:not(.dragging):not(.placeholder)')];
  return cells.reduce((closest, child) => {
    const box = child.getBoundingClientRect();
    const offset = x - box.left - box.width / 2;
    if (offset < 0 && offset > closest.offset) {
      return { offset, element: child };
    }
    return closest;
  }, { offset: Number.NEGATIVE_INFINITY, element: null }).element;
}
```

---

## FLIP Slide Animation (`animateSlide`)

Wraps a DOM mutation so surrounding elements animate into their new positions.

```js
function animateSlide(container, mutate) {
  const cells = [...container.querySelectorAll('.cell')];
  const first = new Map(cells.map(c => [c, c.getBoundingClientRect()])); // snapshot BEFORE
  mutate();
  cells.forEach(cell => {
    const last = cell.getBoundingClientRect();
    const prev = first.get(cell);
    if (!prev) return;
    const dx = prev.left - last.left;
    const dy = prev.top  - last.top;
    if (!dx && !dy) return;
    cell.style.transition = 'none';
    cell.style.transform  = `translate(${dx}px, ${dy}px)`;
    requestAnimationFrame(() => requestAnimationFrame(() => {   // double-rAF to flush
      cell.style.transition = 'transform 0.15s ease-out';
      cell.style.transform  = '';
      cell.addEventListener('transitionend', function clear() {
        cell.style.transition = '';
        cell.removeEventListener('transitionend', clear);
      });
    }));
  });
}
```

Usage inside `dragover`:

```js
histGrid.addEventListener('dragover', e => {
  e.preventDefault();
  e.dataTransfer.dropEffect = 'move';
  if (!placeholder) return;
  const after = getDragAfterElement(histGrid, e.clientX);
  // skip if placeholder is already in the right spot
  if (after === placeholder.nextElementSibling) return;
  if (after == null && placeholder === histGrid.lastElementChild) return;
  animateSlide(histGrid, () => {
    if (after == null) histGrid.appendChild(placeholder);
    else histGrid.insertBefore(placeholder, after);
  });
});
```

---

## Persisting Order

```js
// on drop AND dragend (belt-and-suspenders)
histGrid.addEventListener('drop', e => {
  e.preventDefault();
  commitOrder(histGrid);         // ⚠️ do not omit — dragend may not fire on all browsers
});

function commitOrder(container) {
  const newOrder = [...container.querySelectorAll('.cell')].map(c => c.dataset.key);
  if (newOrder.join('|') !== history.join('|')) {
    history = newOrder;
    saveHistory();
  }
}
```

---

## Errors to Avoid

| ❌ Mistake | ✅ Fix |
|---|---|
| Referencing a `cells` variable that was declared inside the old function and then deleted during refactor | Always declare `cells` **inside** `getDragAfterElement` with a fresh query |
| Excluding only `.dragging` in the selector — but the real cell is now hidden and replaced by `.placeholder` | Exclude **both**: `.cell:not(.dragging):not(.placeholder)` |
| Removing `commitOrder()` from the `drop` handler during a refactor | Keep it in **both** `drop` and `dragend`; `dragend` alone is unreliable cross-browser |
| Calling `cell.style.display = 'none'` synchronously in `dragstart` | Wrap in `setTimeout(..., 0)` — the browser needs one tick to capture the drag image first |
| Animating the hidden dragged cell itself in `animateSlide` | The dragged cell has `display:none` during the drag — `getBoundingClientRect()` returns zeros. Guard with `if (!prev) return` |
| Forgetting `e.dataTransfer.setData(...)` in `dragstart` | Required by Firefox; without it, `drop` never fires |

---

## CSS Checklist

```css
.cell { position: relative; }         /* needed for absolute children (e.g. delete button) */

.cell.dragging    { opacity: 0.4; }   /* visual feedback on the drag image */
.cell.placeholder {                   /* the gap left behind */
  opacity: 0.35;
  border-style: dashed;
}
```

---

## References

See `references/flip-animation.md` for a deeper explanation of the FLIP technique.
