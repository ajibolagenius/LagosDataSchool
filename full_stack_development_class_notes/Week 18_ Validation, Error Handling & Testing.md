# Week 18: Validation, Error Handling & Testing

Flowboard is authenticated and persistent, but it still trusts every request's shape blindly, and its error responses are inconsistent — some routes might crash the server entirely on bad input. This week closes both gaps: real input validation with Zod, one centralized way of turning any error into a clean response, and a real automated test suite that proves the whole API works, not just "seemed fine in Postman."

---

## Module 1: Validation With Zod

### 1. Why Validate at the Edge

* **Lecture & Concepts:**
    * Right now, `POST /boards` with a missing or wrong-typed `title` doesn't fail cleanly — it either silently creates a broken board or crashes somewhere deep in the Model layer with a confusing database error. Validating input **at the edge**, before it enters any business logic, catches this immediately with a clear, actionable error.
    * **Zod** lets you define a schema describing exactly what valid data looks like, and `schema.parse(data)` either returns correctly-typed data or throws a detailed `ZodError`.

### 2. Building Real Schemas

* **In-Depth Example:**
    ```bash
    npm install zod
    ```
    ```ts
    // src/schemas/boardSchema.ts
    import { z } from 'zod';

    export const createBoardSchema = z.object({
      title: z.string().trim().min(1, 'Title is required').max(100),
    });

    export const updateBoardSchema = createBoardSchema.partial();
    ```
    ```ts
    // src/schemas/taskSchema.ts
    export const createTaskSchema = z.object({
      title: z.string().trim().min(1).max(200),
      completed: z.boolean().optional().default(false),
    });
    ```
    * `.partial()` makes every field on an existing schema optional — the correct shape for a `PATCH`, where a client might update just one field.

* **Class Exercise: Write the Real Schemas**
    1. Create `boardSchema.ts`, `listSchema.ts`, and `taskSchema.ts`, each with a `create` schema and a `.partial()` update variant matching your real Drizzle columns.

### 3. The Validation Middleware

* **In-Depth Example:**
    ```ts
    // src/middlewares/validate.ts
    import { Request, Response, NextFunction } from 'express';
    import { ZodSchema } from 'zod';

    export function validate(schema: ZodSchema) {
      return (req: Request, res: Response, next: NextFunction) => {
        const result = schema.safeParse(req.body);
        if (!result.success) {
          return res.status(400).json({
            error: 'Validation failed',
            details: result.error.flatten().fieldErrors,
          });
        }
        req.body = result.data; // now guaranteed correctly-typed and trimmed
        next();
      };
    }
    ```
    ```ts
    // src/routes/boardRoutes.ts
    router.post('/', validate(createBoardSchema), BoardController.createBoard);
    router.patch('/:id', validate(updateBoardSchema), BoardController.updateBoard);
    ```

* **Class Exercise: Validate Every Write**
    1. Build `validate` exactly as shown, and apply it to every `POST`/`PATCH` route across `boards`, `lists`, and `tasks`.
    2. Send a request with a missing `title` via Postman and confirm you get a clean `400` with field-level detail, not a server crash.

---

## Module 2: Centralized Error Handling

### 1. Custom Error Classes

* **Lecture & Concepts:**
    * Scattering `throw new Error('...')` everywhere loses information about *what kind* of failure it was, which is exactly what determines the right HTTP status code. A small hierarchy of custom error classes carries that information along with the error itself.

* **In-Depth Example:**
    ```ts
    // src/errors/CustomError.ts
    export class CustomError extends Error {
      constructor(message: string, public statusCode: number) {
        super(message);
      }
    }

    export class NotFoundError extends CustomError {
      constructor(message = 'Resource not found') { super(message, 404); }
    }

    export class ForbiddenError extends CustomError {
      constructor(message = 'Forbidden') { super(message, 403); }
    }

    export class ValidationError extends CustomError {
      constructor(message = 'Invalid input') { super(message, 400); }
    }
    ```

### 2. One Central Handler

* **Lecture & Concepts:**
    * An Express error-handling middleware has a distinct signature — **four** parameters, `(err, req, res, next)`. Express recognizes this exact shape and routes any error passed to `next(error)` (or thrown inside an `async` handler wrapped correctly) here, instead of to a normal route.

* **In-Depth Example:**
    ```ts
    // src/middlewares/errorHandler.ts
    import { Request, Response, NextFunction } from 'express';
    import { CustomError } from '../errors/CustomError';

    export function errorHandler(err: unknown, req: Request, res: Response, next: NextFunction) {
      console.error(err); // always log — a silent failure in production is worse than a noisy one

      if (err instanceof CustomError) {
        return res.status(err.statusCode).json({ error: err.message });
      }

      res.status(500).json({ error: 'Internal server error' });
    }
    ```
    ```ts
    // src/index.ts — registered LAST, after every route
    app.use(errorHandler);
    ```
    ```ts
    // A controller now looks like this:
    async function updateBoard(req: Request, res: Response, next: NextFunction) {
      try {
        const board = await assertOwnsBoard(Number(req.params.id), req.userId);
        const updated = await BoardModel.update(board.id, req.body.title);
        res.status(200).json(updated);
      } catch (error) {
        next(error); // hands off to errorHandler — no res.status/.json duplicated in every catch
      }
    }
    ```

* **Class Exercise: Wire the Central Handler**
    1. Build the `CustomError` hierarchy and `errorHandler` exactly as shown, registering it last in `src/index.ts`.
    2. Update `assertOwnsBoard` (Week 17) to throw `NotFoundError`/`ForbiddenError` instead of a plain `Error`.
    3. Rewrite every controller's `catch` block to simply `next(error)`, removing duplicated `res.status(...).json(...)` error-shaping logic.

---

## Module 3: Automated Testing

### 1. Why Postman Isn't Enough

* **Lecture & Concepts:**
    * Manually re-clicking through Postman after every change doesn't scale and doesn't run automatically. **Vitest** (the test runner) plus **Supertest** (which sends real HTTP requests directly to your Express app, in-process, without needing a running server) together give you a real, repeatable integration test suite.

### 2. A Disposable Test Database

* **Lecture & Concepts:**
    * Tests that write real data need their own database, never your development one — a second free Neon project (or a Neon **branch**, which is a full, isolated copy) works well, pointed to by a separate `TEST_DATABASE_URL`.

### 3. Writing Real Integration Tests

* **In-Depth Example:**
    ```bash
    npm install -D vitest supertest
    ```
    ```ts
    // src/app.test.ts
    import { describe, it, expect, beforeAll } from 'vitest';
    import request from 'supertest';
    import { app } from './app'; // app exported separately from the app.listen() call

    describe('Flowboard API', () => {
      let cookie: string;

      beforeAll(async () => {
        await request(app).post('/api/v1/auth/signup').send({
          email: 'test@example.com',
          password: 'a-real-password-123',
        });
        const res = await request(app).post('/api/v1/auth/login').send({
          email: 'test@example.com',
          password: 'a-real-password-123',
        });
        cookie = res.headers['set-cookie'][0];
      });

      it('rejects an unauthenticated request', async () => {
        const res = await request(app).get('/api/v1/boards');
        expect(res.status).toBe(401);
      });

      it('creates and lists a board for the authenticated user', async () => {
        const createRes = await request(app)
          .post('/api/v1/boards')
          .set('Cookie', cookie)
          .send({ title: 'Personal Projects' });
        expect(createRes.status).toBe(201);

        const listRes = await request(app).get('/api/v1/boards').set('Cookie', cookie);
        expect(listRes.status).toBe(200);
        expect(listRes.body.some((b: { title: string }) => b.title === 'Personal Projects')).toBe(true);
      });

      it('rejects creating a board with no title', async () => {
        const res = await request(app).post('/api/v1/boards').set('Cookie', cookie).send({});
        expect(res.status).toBe(400);
      });
    });
    ```
    * Splitting `app` (the Express app, exported) from the `app.listen(...)` call (kept only in a separate `server.ts` entrypoint) is what lets Supertest exercise real routes without an actual open network port.

* **Class Exercise: Cover the Real API**
    1. Split `index.ts` into `app.ts` (exports the configured `app`) and `server.ts` (imports `app`, calls `.listen()`).
    2. Point `DATABASE_URL` at a disposable test database when running tests (a separate `.env.test`, loaded only in the test setup).
    3. Write the tests above, plus one covering the full `boards` → `lists` → `tasks` creation chain, and one confirming a second user genuinely can't access the first user's board (`403`).
    4. Run `npx vitest` and confirm everything passes.

---

### Week 18: Comprehensive Assignment

**Objective:** *(Milestone: Flowboard API v1 — authenticated, validated, tested — complete.)*

**Requirements:**

1. **Validation:** Every write endpoint across `boards`, `lists`, and `tasks` validated by a Zod schema, returning clean `400`s with field-level detail on failure.
2. **Error Handling:** A `CustomError` hierarchy and one centralized `errorHandler`; every controller delegates errors via `next(error)` instead of hand-shaping responses.
3. **Tests:** A real Vitest + Supertest suite covering signup/login, unauthenticated rejection, full CRUD for at least `boards`, and one cross-user ownership check — run against a disposable test database.
4. **All Green:** `npx vitest` passes with zero failures.
5. **Git:** Committed and pushed via the branch → PR → merge workflow.

**Bonus Challenge:** Add a test that deliberately sends a `title` over 100 characters and confirms Zod's `.max(100)` rejects it — proving your validation schema's boundary conditions, not just the happy path.
