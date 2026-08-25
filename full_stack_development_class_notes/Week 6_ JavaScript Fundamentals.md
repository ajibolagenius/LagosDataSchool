# Week 6: JavaScript Fundamentals

Lagos Spots has real structure and real style, but right now it's a static document — the dark-mode button from last week works only because of one small hand-written script, and everything else just sits there. This week you write your first substantial JavaScript: real variables, real logic, and the beginning of rendering the spots list from *data* instead of hardcoded HTML.

---

## Module 1: Meeting JavaScript

### 1. What JavaScript Actually Does

* **Lecture & Concepts:**
    * If HTML is the page's content and CSS is its appearance, JavaScript is what senses what a visitor does and reacts to it: a click, a typed search term, a submitted form. Without it, a page can only ever show what was written into the HTML ahead of time.
    * JavaScript runs **client-side** — the code is downloaded to the visitor's browser and executes right there, which is why it can react instantly without waiting on a server round-trip for every click.

* **Class Exercise: Find the Gap**
    1. Click your dark-mode toggle from Week 5 and confirm it still works — that's the only JS in the project so far.
    2. Try clicking anywhere else on the page (a category tag, a card). Confirm nothing happens yet — that's exactly what this week changes.

---

### 2. Linking JavaScript the Right Way

* **Lecture & Concepts:**
    * Same three options as CSS had: **external** (a separate `.ts`/`.js` file, linked — the standard), **internal** (a `<script>` block in the HTML — occasionally fine for a tiny prototype), and **inline** (`onclick="..."` directly on a tag — avoid entirely, it mixes structure and behavior).
    * Historically, scripts were placed just before `</body>` so the HTML existed before JS looked for it. The modern approach is a `<script defer>` tag in `<head>`: the browser downloads it in the background while still parsing the HTML, and only runs it once parsing finishes — fast, and keeps every `<script>` tag in one predictable place.

* **In-Depth Example:**
    ```html
    <head>
      <script type="module" defer src="/src/main.ts"></script>
    </head>
    ```

* **Class Exercise: Confirm the Link**
    1. In `main.ts` (already scaffolded by Vite from Week 5), add `console.log('Lagos Spots main.ts loaded');` as the first line.
    2. Refresh the page, open the console, and confirm the message appears exactly once.

---

### 3. The Console & Debugging

* **Lecture & Concepts:**
    * `console.log()` prints a value so you can see what your code is actually doing — you'll use it constantly.
    * `console.warn()` and `console.error()` print in yellow/red respectively, useful for flagging real problems distinctly from routine debugging output.
    * Beyond logging: **breakpoints**. In DevTools' **Sources** tab, click a line number to pause execution there the next time it runs — then hover any variable to inspect its live value, or step through line-by-line. This is the same skill you'll rely on to inspect a real API response in Week 9.

* **Class Exercise: Set a Real Breakpoint**
    1. In `main.ts`, write a small function that logs a message when called, and call it once.
    2. In the Sources tab, set a breakpoint on the `console.log` line inside that function, refresh, and confirm execution pauses there.

---

### 4. Linting & Formatting

* **Lecture & Concepts:**
    * **ESLint** catches likely mistakes and enforces consistent patterns before they become bugs — a missed `===`, an unused variable. **Prettier** auto-formats code so nobody argues about spacing or quotes. Turning both on now, before much code exists, builds the habit early.

* **Class Exercise: Turn On the Safety Net**
    1. Install the ESLint and Prettier extensions in VS Code.
    2. Enable "Format on Save" in VS Code's settings.
    3. Deliberately write `if (1 == 1) { }` in `main.ts` and confirm ESLint flags the `==`.

---

## Module 2: Variables, Types & Operators

### 1. `let` and `const`

* **Lecture & Concepts:**
    * `const`: the default choice, used roughly 90% of the time. Must be assigned immediately, can never be reassigned.
    * `let`: only when a value genuinely needs to change later — a running count, a toggle.
    * `var` is not used in modern code — its scoping rules "leak" out of blocks in ways `let`/`const` were specifically designed to fix.

* **In-Depth Example (Real Search-Bar Elements):**
    ```ts
    const searchInput = document.querySelector<HTMLInputElement>('#search-input');
    const spotsList = document.querySelector<HTMLElement>('#spots-list');

    let currentCategory = 'all'; // this will change as the user clicks filters
    ```

### 2. Primitive Types

* **Lecture & Concepts:**
    * JavaScript is dynamically typed — a variable's type is whatever value currently lives in it. The primitives to know: **string**, **number**, **boolean**, **undefined** (declared but never assigned — usually accidental), and **null** (deliberately empty, set on purpose by you).
    * `typeof` inspects a value's type at runtime: `typeof currentCategory` → `"string"`.

### 3. Template Literals

* **Lecture & Concepts:**
    * Joining strings with `+` is easy to get wrong (`"5" + 10` produces `"510"`, not `15` — a classic type-coercion trap). **Template literals**, written with backticks and `${...}`, embed variables directly and read far more clearly.

* **In-Depth Example:**
    ```ts
    const name = 'Terra Kulture';
    const category = 'food';

    const label = `${name} — filed under ${category}`;
    ```

### 4. Operators

* **Lecture & Concepts:**
    * **Comparison:** `===` (strict equality — always use this) checks both value and type; `==` (loose equality — avoid) silently converts types first, hiding real bugs (`5 == "5"` is `true`).
    * **Logical:** `&&` (both sides must be truthy), `||` (at least one side truthy), `!` (flips a boolean).
    * **Nullish coalescing (`??`):** like `||`, but only falls back when the left side is specifically `null` or `undefined` — unlike `||`, it won't wrongly treat `0` or `""` as "missing." Useful for a real default: `const category = savedCategory ?? 'all';` correctly keeps a saved value of `''` if that were ever valid, where `||` would incorrectly discard it.

* **Class Exercise: Practice on Real Values**
    1. In the console, declare `const price = 0;` and log `price || 'Free'` vs. `price ?? 'Free'` — note they differ, and explain why in a comment.
    2. Declare two variables holding the same number as a `string` and as a `number`, and confirm `===` correctly reports them as different while `==` does not.

---

## Module 3: Control Flow

### 1. Conditionals

* **Lecture & Concepts:**
    * `if`/`else` branches based on a condition. The ternary operator (`condition ? valueIfTrue : valueIfFalse`) is a compact expression form, best for simple either/or values rather than multi-step logic. `switch` is useful when comparing one value against many fixed possibilities.

* **In-Depth Example (The Real Category Filter Decision):**
    ```mermaid
    flowchart TD
        A[User clicks a category filter] --> B{category === 'all'?}
        B -->|Yes| C[Show every spot]
        B -->|No| D[Show only spots<br/>where spot.category === category]
    ```
    ```ts
    function getVisibleSpots(spots: { category: string }[], category: string) {
      if (category === 'all') {
        return spots;
      }
      return spots.filter((spot) => spot.category === category);
    }
    ```
    *(`.filter()` itself is a Week 7 topic — for now, focus on the `if`/`return` logic; treat the one-liner above as a preview.)*

* **Class Exercise: Implement the Decision**
    1. Write `getVisibleSpots` exactly as shown, treating `.filter()` as a black box for now.
    2. Test it in the console with a small hardcoded array of `{ category: string }` objects and both `'all'` and a real category name.

---

## Module 4: Functions & Scope

### 1. Three Ways to Write a Function

* **Lecture & Concepts:**
    * **Function declaration:** `function renderSpots(spots) { ... }` — hoisted (usable before its definition in the file).
    * **Function expression:** `const renderSpots = function(spots) { ... };` — not hoisted.
    * **Arrow function:** `const renderSpots = (spots) => { ... };` — the modern default for short, self-contained functions; also handles `this` differently, which will matter once you reach class-based patterns later in the course.

### 2. Block Scope

* **Lecture & Concepts:**
    * A variable declared with `let`/`const` only exists inside the nearest enclosing `{ }` block — its **scope**. A variable declared inside an `if` block or a function body isn't visible outside it. This is what makes `let`/`const` safer than `var`, which ignores block boundaries entirely.

* **In-Depth Example (The Real Render Function):**
    ```ts
    function renderSpots(spots: { name: string; category: string }[]) {
      if (!spotsList) return;

      spotsList.innerHTML = spots
        .map((spot) => `<article class="spot-card"><h3>${spot.name}</h3></article>`)
        .join('');
    }
    ```

* **Class Exercise: Build `renderSpots`**
    1. Write a real `spots` array of at least 5 objects (`{ name, category, area, description }`), matching your Week 2 cards' content.
    2. Write `renderSpots(spots)` as shown, and call it once on page load so the grid renders from this array instead of hardcoded HTML.
    3. Confirm the page still looks correct — the *content* now comes from JS, even though nothing about the visual result changed yet.

---

### Week 6: Comprehensive Assignment

**Objective:** Give Lagos Spots its first real logic: data-driven rendering and a working category filter.

**Requirements:**

1. **Linking:** `main.ts` linked with `<script type="module" defer>`; logs confirm it loads.
2. **Data:** A real `spots` array of at least 5 objects with `name`, `category`, `area`, and `description`.
3. **Rendering:** `renderSpots(spots)` builds the spots grid's HTML from that array — no hardcoded `.spot-card` markup left in `index.html`.
4. **Filtering:** `getVisibleSpots(spots, category)` implements the decision-flow diagram above; clicking a category button (from your Week 2/5 tag chips, wired with a real click handler) calls `renderSpots(getVisibleSpots(spots, category))`.
5. **Strict Equality:** `===` used throughout; no `==` anywhere in the file (confirm ESLint has nothing to flag).
6. **Git:** Committed and pushed via the branch → PR → merge workflow.

**Bonus Challenge:** Build a small "Price Estimator" as standalone console practice, unconnected to the main app: given a `const basePrice = 0;`, log both `basePrice || 'Contact for pricing'` and `basePrice ?? 'Contact for pricing'`, and write a one-sentence comment explaining why a real free spot (price `0`) should use `??`, not `||`.
