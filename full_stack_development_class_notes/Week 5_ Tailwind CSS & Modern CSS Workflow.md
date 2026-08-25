# Week 5: Tailwind CSS & Modern CSS Workflow

You've now written enough hand-rolled CSS to understand exactly what a stylesheet does — the box model, layout, specificity, all of it earned the hard way. This week you meet the tool most production teams reach for instead: **Tailwind CSS**, a utility-first framework that replaces most hand-written CSS with small, composable class names. You'll also meet a real build tool, **Vite**, for the first time — the same tool you'll use for React starting in Part III.

---

## Module 1: Why Utility-First CSS, and Setting It Up

### 1. What "Utility-First" Means

* **Lecture & Concepts:**
    * So far, styling a card meant writing a custom class (`.spot-card`) and a separate CSS rule for every property you wanted. **Utility-first CSS** flips this: instead of custom classes, you compose *many small, single-purpose classes* directly in your HTML — `p-4` (padding), `flex`, `rounded-lg`, `text-lg`.
    * The tradeoff is real, not one-sided: you write less CSS and iterate faster, but your HTML carries more classes, and a team needs a shared vocabulary for what each utility does. Most production frontend teams today have settled on this tradeoff being worth it — which is why it's worth learning deliberately, not just as a shortcut.
    * Everything you learned in Weeks 3–4 (box model, Flexbox, Grid, specificity) still applies — Tailwind's utilities are just a fast way to write those exact same CSS properties.

* **Class Exercise: Compare Two Approaches**
    1. Look back at your Week 3 `.spot-card` CSS rule (padding, border, background).
    2. Predict what the equivalent Tailwind utility classes might look like, just from the property names you already know (you'll confirm this in the next section).

---

### 2. Setting Up Tailwind with Vite

* **Lecture & Concepts:**
    * **Vite** is a modern frontend build tool: it serves your project with a fast local dev server and bundles it for production. It replaces manually double-clicking an HTML file.
    * Tailwind needs a build step to scan your files and generate only the CSS you actually use — this is exactly what Vite provides.

* **In-Depth Example (Terminal):**
    ```bash
    npm create vite@latest lagos-spots-web -- --template vanilla-ts
    cd lagos-spots-web
    npm install
    npm install tailwindcss @tailwindcss/vite
    ```
    ```ts
    // vite.config.ts
    import { defineConfig } from 'vite'
    import tailwindcss from '@tailwindcss/vite'

    export default defineConfig({
      plugins: [tailwindcss()],
    })
    ```
    ```css
    /* src/style.css */
    @import "tailwindcss";
    ```

* **Class Exercise: Migrate the Project**
    1. Scaffold a new Vite project as shown above (choosing the `vanilla-ts` template gets TypeScript scaffolding ready early, even though you won't write TS syntax until Week 10).
    2. Move your existing `index.html`, `submit.html`, and `assets/` into this new project.
    3. Replace your old `<link>` tags with the single Tailwind import shown above, and run `npm run dev` to confirm the dev server starts.

---

## Module 2: Core Utilities

### 1. Spacing, Flex & Grid Utilities

* **Lecture & Concepts:**
    * Spacing utilities follow a consistent scale: `p-4` (padding), `m-4` (margin), `gap-4` — the number is a multiple of Tailwind's base spacing unit (`4` = 1rem).
    * Layout utilities map directly onto what you already know: `flex`, `flex-col`, `justify-between`, `items-center`, `grid`, `grid-cols-3`.

* **In-Depth Example (The Real Spot Card, in Tailwind):**
    ```html
    <article class="bg-white p-4 rounded-lg border border-gray-200 hover:shadow-md transition">
      <div class="flex justify-between items-start gap-2">
        <h3 class="text-lg font-semibold">Terra Kulture</h3>
        <span class="text-xs bg-orange-100 text-orange-700 px-2 py-1 rounded-full">Art</span>
      </div>
      <p class="text-sm text-gray-600 mt-2">Art gallery, restaurant, and live music space.</p>
    </article>
    ```

* **Class Exercise: Rebuild One Card**
    1. Pick one existing `.spot-card` and rebuild it using only Tailwind utility classes, matching the example's shape.
    2. Compare it side by side with an un-migrated card — confirm they look the same.

---

### 2. Responsive Prefixes & Dark Mode

* **Lecture & Concepts:**
    * Every utility can be prefixed with a breakpoint to apply only from that width up — `md:flex-row` means "apply `flex-row` from the `md` breakpoint (768px) upward," directly replacing a hand-written `@media (min-width: 768px)` block.
    * Tailwind's `dark:` variant applies a utility only when dark mode is active. With the `class` strategy, dark mode toggles by adding/removing a `dark` class on `<html>` — giving you full control via a button click, rather than only following the OS setting.

* **In-Depth Example:**
    ```html
    <header class="flex flex-col md:flex-row md:justify-between md:items-center gap-3 p-4 bg-white dark:bg-gray-900">
      <h1 class="text-xl font-bold dark:text-white">Lagos Spots</h1>
    </header>
    ```
    ```ts
    // src/main.ts
    document.getElementById('theme-toggle')?.addEventListener('click', () => {
      document.documentElement.classList.toggle('dark');
    });
    ```

* **Class Exercise: Responsive Header + Dark Mode Toggle**
    1. Rebuild the header with `flex-col md:flex-row` and the responsive utilities above.
    2. Add a `<button id="theme-toggle">🌓</button>` to the header, and wire up the toggle script above in `main.ts`.
    3. In `tailwind.config`, confirm (or set) the dark mode strategy to `class`, then test clicking the button.

---

## Module 3: Theming & Component Patterns

### 1. Custom Theme Configuration

* **Lecture & Concepts:**
    * Tailwind's default color palette is a solid start, but a real product usually wants its own named colors. A theme config lets you define e.g. a distinct color per spot category, then reuse a short, meaningful name everywhere instead of a raw hex code.

* **In-Depth Example:**
    ```css
    /* src/style.css */
    @import "tailwindcss";

    @theme {
      --color-food: #f5a623;
      --color-study: #4a90d9;
      --color-nightlife: #9b51e0;
      --color-events: #27ae60;
    }
    ```
    ```html
    <span class="bg-food/10 text-food px-2 py-1 rounded-full text-xs">Food</span>
    ```

* **Class Exercise: Theme the Category Badges**
    1. Add the four category colors to your theme config.
    2. Update every category badge across your cards to use the matching themed color, based on that spot's real category.

---

### 2. When to Extract a Repeated Pattern

* **Lecture & Concepts:**
    * Repeating the same long string of utility classes across every card is normal and expected at this stage — real component extraction (a proper `SpotCard` you define once and reuse) is what React gives you starting in Part III. For now, if a class combination repeats *identically* many times, Tailwind's `@apply` can pull it into one custom class — but use this sparingly, since it re-introduces the specificity/naming tradeoffs utility classes were meant to avoid.

* **In-Depth Example:**
    ```css
    .badge {
      @apply text-xs px-2 py-1 rounded-full;
    }
    ```

* **Class Exercise: One Careful Extraction**
    1. Extract just the shared, non-color part of your badge classes into a single `.badge` class using `@apply`.
    2. Confirm every badge still looks correct, now combining `.badge` with its category-specific color utility.

---

### Week 5: Comprehensive Assignment

**Objective:** A fully Tailwind-powered Lagos Spots, running on Vite, with a working dark mode and themed category badges.

**Requirements:**

1. **Setup:** Project runs on Vite with Tailwind installed via `@tailwindcss/vite`; old hand-written CSS files removed.
2. **Migration:** Every existing element (header, spot cards, form) is restyled using Tailwind utilities, visually matching or improving on the Week 3–4 version.
3. **Responsive:** At least the header uses a responsive prefix (`md:`) to change layout at a real breakpoint.
4. **Dark Mode:** A working toggle button that adds/removes the `dark` class on `<html>`, with `dark:` variants applied to at least the header and body background.
5. **Theming:** A custom theme config with one named color per spot category, used consistently across all badges.
6. **Git:** Committed and pushed via the branch → PR → merge workflow.

**Bonus Challenge:** Persist the dark mode choice to `localStorage` so it survives a page refresh — you'll only need `localStorage.getItem`/`setItem` and a small check on page load, both of which you'll formally learn in Week 7, so treat this as a preview.
