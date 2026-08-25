# Week 14: Advanced React, Testing & Ship

This is the last week of the React phase, and it closes out the frontend running project properly: extracting reusable logic into custom hooks, avoiding wasted re-renders as the app grows, writing real automated tests instead of just clicking around by hand, and deploying the finished app to a real, public URL.

---

## Module 1: Custom Hooks & Performance

### 1. Extracting a Custom Hook

* **Lecture & Concepts:**
    * Any time logic built from `useState`/`useEffect` is duplicated — or even just conceptually "one thing" — it's a candidate for a **custom hook**: a regular function, prefixed `use`, that calls other hooks internally and returns whatever a component needs.
    * `useDebouncedValue` from Week 12 was already a small custom hook. This week you extract the *entire* search flow into one.

* **In-Depth Example:**
    ```tsx
    // src/hooks/useSpotSearch.ts
    import { useState } from 'react';
    import { useQuery } from '@tanstack/react-query';
    import { searchLocations } from '../api';
    import { useDebouncedValue } from './useDebouncedValue';

    export function useSpotSearch() {
      const [query, setQuery] = useState('');
      const debouncedQuery = useDebouncedValue(query, 300);

      const { data: results = [], isLoading, isError } = useQuery({
        queryKey: ['locationSearch', debouncedQuery],
        queryFn: () => searchLocations(debouncedQuery),
        enabled: debouncedQuery.length > 0,
      });

      return { query, setQuery, results, isLoading, isError };
    }
    ```
    ```tsx
    // SearchBar.tsx shrinks to almost nothing:
    function SearchBar() {
      const { query, setQuery, results, isLoading, isError } = useSpotSearch();
      return (
        <div>
          <input value={query} onChange={(e) => setQuery(e.target.value)} />
          {isLoading && <p>Searching...</p>}
          {isError && <p>Something went wrong.</p>}
        </div>
      );
    }
    ```

* **Class Exercise: Extract `useSpotSearch`**
    1. Move Week 13's search logic out of `SearchBar` and into `useSpotSearch.ts` exactly as shown.
    2. Confirm `SearchBar` still works identically, now as a much smaller, purely presentational component.

---

### 2. Avoiding Unnecessary Re-Renders

* **Lecture & Concepts:**
    * When a parent re-renders, every child re-renders by default — usually fine, but occasionally wasteful for an expensive component.
    * `React.memo(Component)` skips re-rendering a component if its props haven't actually changed.
    * `useMemo(fn, deps)` caches an expensive **computed value** between renders, recalculating only when a dependency changes.
    * `useCallback(fn, deps)` caches a **function reference** itself — useful specifically when that function is passed as a prop to a `memo`-wrapped child, since a freshly-created function on every render would defeat the memoization.

* **In-Depth Example:**
    ```tsx
    import { memo, useMemo } from 'react';

    const SpotCard = memo(function SpotCard({ spot, onToggleFavorite }: SpotCardProps) {
      // unchanged from Week 11/13 — wrapping in memo is the only change here
    });
    ```
    ```tsx
    // In HomePage.tsx
    const visibleSpots = useMemo(
      () => (category === 'all' ? spots : spots.filter((s) => s.category === category)),
      [spots, category]
    );
    ```

* **Class Exercise: Apply Memoization Deliberately**
    1. Wrap `SpotCard` in `React.memo`.
    2. Memoize the filtered `visibleSpots` calculation with `useMemo`, exactly as shown.
    3. Using the React DevTools "Highlight updates when components render" setting, confirm unrelated state changes (like typing in the search bar) no longer re-render every `SpotCard`.

---

### 3. Code-Splitting a Route

* **Lecture & Concepts:**
    * By default, your entire app ships as one JavaScript bundle, downloaded before anything renders. `React.lazy()` + `<Suspense>` split a component into its own chunk, downloaded only when that route is actually visited — genuinely useful for a route like `SpotDetailPage` that most visitors on the home page will never open.

* **In-Depth Example:**
    ```tsx
    import { lazy, Suspense } from 'react';

    const SpotDetailPage = lazy(() => import('./pages/SpotDetailPage'));

    // in the route config:
    {
      path: 'spots/:id',
      element: (
        <Suspense fallback={<p>Loading...</p>}>
          <SpotDetailPage />
        </Suspense>
      ),
    }
    ```

* **Class Exercise: Lazy-Load the Detail Page**
    1. Convert `SpotDetailPage`'s import to `React.lazy`, wrapped in `<Suspense>`.
    2. In your browser's Network tab, confirm a separate chunk downloads only the first time you navigate to a `/spots/:id` URL.

---

## Module 2: Testing With Vitest & React Testing Library

### 1. Testing Behavior, Not Implementation

* **Lecture & Concepts:**
    * **React Testing Library (RTL)** deliberately tests what a *user* sees and does — rendering a component, finding elements the way a user would (by visible text, label, or role), and simulating real interaction — rather than reaching into a component's internal state directly. This makes tests survive internal refactors that don't change actual behavior.
    * **Vitest** is the test runner (similar to Jest, but built to integrate directly with Vite).

* **In-Depth Example:**
    ```bash
    npm install -D vitest @testing-library/react @testing-library/user-event jsdom
    ```
    ```tsx
    // src/components/SpotCard.test.tsx
    import { render, screen } from '@testing-library/react';
    import userEvent from '@testing-library/user-event';
    import { describe, it, expect, vi } from 'vitest';
    import SpotCard from './SpotCard';

    describe('SpotCard', () => {
      it('calls onToggleFavorite when the favorite button is clicked', async () => {
        const handleToggle = vi.fn();
        render(
          <SpotCard
            id="terra-kulture"
            name="Terra Kulture"
            category="food"
            area="Victoria Island"
            description="Art gallery and restaurant."
            favorited={false}
            onToggleFavorite={handleToggle}
          />
        );

        await userEvent.click(screen.getByRole('button', { name: /favorite/i }));

        expect(handleToggle).toHaveBeenCalledWith('terra-kulture');
      });
    });
    ```

* **Class Exercise: Test the Real Components**
    1. Set up Vitest + RTL as shown.
    2. Write the `SpotCard` test above (adjust props to match your real interface), and a second test confirming `SubmitSpotForm` shows the inline error when the description is too short.
    3. Run `npx vitest` and confirm both pass.

---

## Module 3: Ship It

### 1. Environment Variables & Building

* **Lecture & Concepts:**
    * Vite exposes environment variables prefixed `VITE_` to your client code via `import.meta.env.VITE_*` — anything without that prefix is intentionally kept out of the client bundle, since client-side code is always publicly visible.
    * `npm run build` compiles the app into a `dist/` folder of static, optimized files — this is what actually gets deployed, not your source files directly.

### 2. Deploying to Vercel

* **Lecture & Concepts:**
    * Vercel connects directly to a GitHub repository: every push to `main` triggers a new production deployment, and every pull request gets its own preview URL automatically — real CI/CD, not a manual upload.

* **Class Exercise: Deploy for Real**
    1. Push `lagos-spots-react` to GitHub if you haven't already.
    2. Create a free Vercel account, import the repository, and deploy it with default settings (Vercel auto-detects Vite).
    3. Open the live URL on your phone and confirm it works end to end: filtering, real search, submitting a spot, and favorites persisting after a refresh.

---

### Week 14: Comprehensive Assignment

**Objective:** *(Milestone: Lagos Spots v2 — React + TS SPA, tested and deployed live.)*

**Requirements:**

1. **Custom Hook:** `useSpotSearch` extracted and used by a simplified `SearchBar`.
2. **Performance:** `SpotCard` wrapped in `React.memo`; the filtered spots list memoized with `useMemo`.
3. **Code-Splitting:** `SpotDetailPage` lazy-loaded with a `<Suspense>` fallback.
4. **Tests:** Real, passing Vitest + RTL tests for `SpotCard`'s favorite toggle and `SubmitSpotForm`'s validation.
5. **Deployment:** The app is live on Vercel, fully functional on a real device, not just your laptop.
6. **Git:** Committed and pushed via the branch → PR → merge workflow.

**Bonus Challenge:** Add a GitHub Pull Request for one small change and confirm Vercel automatically comments with a working preview URL for that PR — a first, real taste of the CI/CD workflow covered in full in Part V.
