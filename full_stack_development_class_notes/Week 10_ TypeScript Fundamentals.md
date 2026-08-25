# Week 10: TypeScript Fundamentals

Your project has been sitting in `.ts` files since Week 5, but you've been writing plain JavaScript inside them — a small type annotation here and there, nothing enforced. This week that changes for real: every function, every piece of data, and especially that unpredictable Nominatim API response gets an actual type, checked by the compiler before your code ever runs.

---

## Module 1: Why TypeScript, and the Basics

### 1. What Static Types Actually Buy You

* **Lecture & Concepts:**
    * JavaScript only tells you about a type mistake — passing a `string` where a function expects a `number`, say — **at runtime**, possibly in front of a real user. **TypeScript** adds a type-checking layer *on top of* JavaScript that catches these mistakes **while you're writing the code**, in your editor, before it ever runs.
    * TypeScript is not a different language — it's JavaScript, plus optional type annotations, compiled down to plain JavaScript. Anything you already know about JS still applies exactly as-is.
    * Most real, production frontend and backend teams today default to TypeScript — it's less a "nice extra" than the expected baseline in a professional codebase.

### 2. Basic Types & Inference

* **Lecture & Concepts:**
    * TypeScript can often **infer** a type without you writing it: `const name = 'Terra Kulture';` is automatically known as `string`. You only need explicit annotations where TypeScript can't guess — usually function parameters.
    * The types you'll use constantly: `string`, `number`, `boolean`, arrays (`string[]` or `Array<string>`), and object shapes.

* **In-Depth Example:**
    ```ts
    const spotCount: number = 5;      // annotation not needed here — inferred already
    const spotName = 'Terra Kulture'; // inferred as string, no annotation needed

    function logSpot(name: string, area: string): void {
      console.log(`${name} is in ${area}`);
    }
    ```

### 3. `interface` for Real Object Shapes

* **Lecture & Concepts:**
    * An `interface` describes the *shape* an object must have — which properties, and what type each one is. Once defined, TypeScript checks every place you create or use that shape against it.
    * `interface` and `type` overlap significantly for object shapes; the convention this course follows is `interface` for object shapes that might be extended, and `type` for unions, primitives, or combinations (covered next module).

* **In-Depth Example (The Real `Spot` Shape):**
    ```ts
    interface Spot {
      id: string;
      name: string;
      category: string;
      area: string;
      description: string;
      favorited: boolean;
      hours: {
        open: string;
        close: string;
      };
    }
    ```

* **Class Exercise: Define and Apply `Spot`**
    1. Create `src/types.ts` and define the `Spot` interface exactly as shown.
    2. In `main.ts`, annotate your existing `spots` array as `Spot[]`.
    3. Deliberately misspell a property name in one spot object (e.g. `catgory` instead of `category`) and confirm TypeScript flags it immediately in your editor — this is the whole point.

---

## Module 2: Typing Functions & a Real, Messy API Response

### 1. Typed Function Parameters & Return Types

* **Lecture & Concepts:**
    * Once `Spot` exists, every function that touches spot data should say so explicitly — this is where TypeScript starts actually preventing real bugs, not just documenting intent.

* **In-Depth Example:**
    ```ts
    function findSpotById(spots: Spot[], id: string): Spot | undefined {
      return spots.find((spot) => spot.id === id);
    }

    function toggleFavorite(spots: Spot[], id: string): Spot[] {
      return spots.map((spot) =>
        spot.id === id ? { ...spot, favorited: !spot.favorited } : spot
      );
    }
    ```
    * `Spot | undefined` is a **union type** — the return value is either a real `Spot`, or `undefined` if nothing matched. TypeScript now forces every caller to actually handle the "not found" case instead of assuming success.

* **Class Exercise: Type Your Existing Functions**
    1. Add parameter and return types to `findSpotById`, `toggleFavorite`, `sortByName`, and `countByCategory` from Week 7.
    2. Wherever a function returns `Spot | undefined`, update its callers to check for `undefined` before using the result (an `if` check, or optional chaining).

---

### 2. `unknown` vs. `any`, and Taming a Real External API

* **Lecture & Concepts:**
    * `any` turns off type checking entirely for that value — it's occasionally unavoidable, but overusing it defeats the purpose of using TypeScript at all.
    * `unknown` is the safer alternative for genuinely unpredictable data (like a raw fetch response): TypeScript *forces* you to narrow or validate it before you're allowed to use it as anything specific.
    * The right pattern for an external API: type the **raw** shape you actually receive separately from your **internal** `Spot` shape, and write one explicit function that converts between them — exactly what `toSpotFromResult` already does conceptually from Week 9, now made type-safe.

* **In-Depth Example:**
    ```ts
    interface NominatimResult {
      display_name: string;
      lat: string;
      lon: string;
      name?: string;
      address?: {
        suburb?: string;
        city?: string;
      };
    }

    function toSpotFromResult(result: NominatimResult): Spot {
      return {
        id: crypto.randomUUID(),
        name: result.name ?? result.display_name.split(',')[0],
        area: result.address?.suburb ?? result.address?.city ?? 'Lagos',
        category: 'unverified',
        description: result.display_name,
        favorited: false,
        hours: { open: '00:00', close: '23:59' },
      };
    }
    ```
    * Notice every field on `NominatimResult` that isn't guaranteed is marked optional (`?`) — this is honest about a real, external API you don't control, rather than pretending it's always complete.

* **Class Exercise: Type the Whole API Layer**
    1. Define `NominatimResult` in `types.ts`, matching the real shape you inspected in Week 9's console output.
    2. Type `searchLocations`'s return value as `Promise<NominatimResult[]>`.
    3. Update `toSpotFromResult` to match the typed version above.

### 3. A First Generic: A Typed `localStorage` Helper

* **Lecture & Concepts:**
    * A **generic** lets a function work with a type that's decided *when it's called*, rather than one fixed type. `<T>` is a placeholder the caller fills in.

* **In-Depth Example:**
    ```ts
    function getStored<T>(key: string, fallback: T): T {
      const raw = localStorage.getItem(key);
      return raw ? (JSON.parse(raw) as T) : fallback;
    }

    const favoriteIds = getStored<string[]>('lagos-spots:favorite-ids', []);
    ```

* **Class Exercise: Generalize Your Storage Helper**
    1. Replace your Week 7 `loadFavoriteIds` with `getStored<string[]>(...)`, as shown.
    2. Confirm your editor now knows `favoriteIds` is a real `string[]`, with autocomplete for array methods.

---

## Module 3: Strict Mode & Finishing the Conversion

### 1. `tsconfig.json` and `strict: true`

* **Lecture & Concepts:**
    * `strict: true` in `tsconfig.json` turns on a whole family of stricter checks at once — including `strictNullChecks`, which stops `null`/`undefined` from silently being treated as any other value. This is the setting that makes the `Spot | undefined` pattern from Module 2 actually enforced, rather than optional.
    * Real production TypeScript codebases run with `strict: true` from day one — retrofitting it onto a large, loose codebase later is far more painful than starting strict.

* **Class Exercise: Turn On Strict Mode**
    1. In `tsconfig.json`, set `"strict": true`.
    2. Run `npx tsc --noEmit` (type-check without producing output files) and work through every error it reports, one at a time.

### 2. Typing DOM Elements & Event Handlers

* **Lecture & Concepts:**
    * `document.querySelector` returns a generic `Element | null` unless you tell it otherwise. Passing the real element type as a generic argument gives you the specific properties that type has (like `.value` on an input).

* **In-Depth Example:**
    ```ts
    const searchInput = document.querySelector<HTMLInputElement>('#search-input');

    searchInput?.addEventListener('input', (event: Event) => {
      const target = event.target as HTMLInputElement;
      handleSearch(target.value);
    });
    ```

* **Class Exercise: Finish the Conversion**
    1. Add correct element-type generics to every `querySelector` call in the codebase.
    2. Type every event handler's parameter, casting `event.target` to the correct element type where needed.
    3. Re-run `npx tsc --noEmit` until it reports zero errors.

---

### Week 10: Comprehensive Assignment

**Objective:** *(Milestone: Lagos Spots v1 — complete.)* A fully typed, strict-mode-clean codebase — every function, every piece of data, the external API response, and every DOM interaction.

**Requirements:**

1. **Types Defined:** `Spot` and `NominatimResult` interfaces exist in `types.ts`, matching your real data shapes.
2. **Typed Functions:** Every function touching spot data has explicit parameter and return types.
3. **Safe API Handling:** `NominatimResult`'s uncertain fields are marked optional, and `toSpotFromResult` handles them with optional chaining/nullish coalescing.
4. **Generics:** At least one generic function (the `getStored<T>` helper) is implemented and used.
5. **Strict Mode:** `"strict": true` is set, and `npx tsc --noEmit` reports zero errors.
6. **Git:** Committed and pushed via the branch → PR → merge workflow.

**Bonus Challenge:** Add a `category` union type (`type Category = 'food' | 'study' | 'nightlife' | 'events' | 'unverified';`) and use it in place of `category: string` everywhere — then deliberately try to assign an invalid category string somewhere and confirm TypeScript rejects it.
