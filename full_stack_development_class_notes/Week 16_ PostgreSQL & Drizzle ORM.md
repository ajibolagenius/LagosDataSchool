# Week 16: PostgreSQL & Drizzle ORM

In Week 15, Flowboard used in-memory arrays that vanished upon server restart. This week, you establish a persistent, relational data foundation using **PostgreSQL** — the industry standard open-source relational database — and **Drizzle ORM**, a modern TypeScript-native Object-Relational Mapper. You will master **Relational Schema Design**, **Foreign Key Cascades**, **ACID Transactions**, **Database Connection Pooling**, and **Type-Safe SQL Query Generation**.

---

## Module 1: Relational Database Architecture & Modeling

### 1. The Core Terminology of Relational Databases

| Term | What It Actually Means | Technical Context |
| :--- | :--- | :--- |
| **RDBMS** | Relational Database Management System (e.g., PostgreSQL, MySQL) that organizes data into tables (relations) of rows and columns. | Guarantees structured schemas, data integrity, and complex multi-table relational joins. |
| **Primary Key (PK)** | A column (or set of columns) whose values uniquely identify every row in a table (`serial` integer or `uuid`). | Indexed automatically with a **B-Tree** index for $O(\log n)$ lookup speed. |
| **Foreign Key (FK)** | A column in one table that references the Primary Key of another table. | Enforces **Referential Integrity** — you cannot insert a Task referencing a non-existent List ID. |
| **Referential Action (`CASCADE`)** | Rules defining what happens to child records when a referenced parent record is deleted or updated. | `ON DELETE CASCADE` automatically deletes all child boards, lists, and tasks when a user account is deleted. |
| **ACID Guarantees** | The four core guarantees of transactional databases: **Atomicity** (all-or-nothing), **Consistency** (valid schema state), **Isolation** (concurrent transactions do not interfere), and **Durability** (committed data survives crashes). | Protects data integrity during multi-step database mutations (e.g., reordering lists or bank transfers). |
| **Connection Pool** | A cache of persistent, reusable TCP database connections managed by the application. | Prevents latency and memory exhaustion caused by opening/closing raw database sockets per request. |

---

### 2. Flowboard Entity-Relationship (ER) Architecture

```mermaid
erDiagram
    USERS ||--o{ BOARDS : "1. owns (1:N)"
    BOARDS ||--o{ LISTS : "2. contains (1:N)"
    LISTS ||--o{ TASKS : "3. contains (1:N)"

    USERS {
        uuid id PK
        varchar email UK "Unique, Not Null"
        text password_hash "Bcrypt Hash"
        timestamp created_at "Default NOW()"
    }

    BOARDS {
        uuid id PK
        uuid user_id FK "References users(id) ON DELETE CASCADE"
        varchar title "Not Null"
        text description
        timestamp created_at "Default NOW()"
    }

    LISTS {
        uuid id PK
        uuid board_id FK "References boards(id) ON DELETE CASCADE"
        varchar title "Not Null"
        integer position "Sort order (0, 1, 2...)"
        timestamp created_at "Default NOW()"
    }

    TASKS {
        uuid id PK
        uuid list_id FK "References lists(id) ON DELETE CASCADE"
        varchar title "Not Null"
        text description
        boolean completed "Default false"
        integer position "Sort order (0, 1, 2...)"
        timestamp created_at "Default NOW()"
    }
```

---

### 3. Under the Hood: Preventing SQL Injection via Parameterized Queries

> [!CAUTION]
> **The SQL Injection Vector:** If user input is concatenated directly into SQL strings (`"SELECT * FROM users WHERE email = '" + req.body.email + "'"`), an attacker can pass `' OR '1'='1` and bypass authentication or drop tables.

Drizzle ORM compiles all queries into **Parameterized SQL Statements**:
- User values are replaced with placeholders (`$1`, `$2`).
- The database engine parses and compiles the query structure *before* binding user parameters as literal data, making SQL injection impossible.

```sql
-- Generated parameterized SQL under the hood:
SELECT id, title, user_id FROM boards WHERE user_id = $1 AND title ILIKE $2;
-- Parameters passed separately: [ 'u-101', '%roadmap%' ]
```

---

## Module 2: Setting Up Drizzle ORM with PostgreSQL

### 1. Installation & Environment Configuration

```bash
# 1. Install Drizzle ORM and PostgreSQL client driver
npm install drizzle-orm pg dotenv

# 2. Install developer CLI migration tool and type definitions
npm install -D drizzle-kit @types/pg
```

#### `.env` File:
```env
PORT=4000
NODE_ENV=development
DATABASE_URL=postgresql://neondb_owner:password@ep-flowboard-123.eu-central-1.aws.neon.tech/flowboard?sslmode=require
```

#### `drizzle.config.ts` (CLI Tooling Config):
```ts
import { defineConfig } from 'drizzle-kit';
import * as dotenv from 'dotenv';
dotenv.config();

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  verbose: true,
  strict: true,
});
```

---

### 2. Relational Schema Declaration (`src/db/schema.ts`)

```ts
import { pgTable, text, timestamp, boolean, integer, uuid, varchar } from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

// 1. Users Table
export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  passwordHash: text('password_hash').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

// 2. Boards Table
export const boards = pgTable('boards', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
  title: varchar('title', { length: 150 }).notNull(),
  description: text('description').default('').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});

// 3. Lists Table
export const lists = pgTable('lists', {
  id: uuid('id').defaultRandom().primaryKey(),
  boardId: uuid('board_id')
    .notNull()
    .references(() => boards.id, { onDelete: 'cascade' }),
  title: varchar('title', { length: 100 }).notNull(),
  position: integer('position').default(0).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

// 4. Tasks Table
export const tasks = pgTable('tasks', {
  id: uuid('id').defaultRandom().primaryKey(),
  listId: uuid('list_id')
    .notNull()
    .references(() => lists.id, { onDelete: 'cascade' }),
  title: varchar('title', { length: 255 }).notNull(),
  description: text('description').default('').notNull(),
  completed: boolean('completed').default(false).notNull(),
  position: integer('position').default(0).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

// ============================================================================
// Drizzle Relations (Used for type-safe nested queries)
// ============================================================================
export const usersRelations = relations(users, ({ many }) => ({
  boards: many(boards),
}));

export const boardsRelations = relations(boards, ({ one, many }) => ({
  user: one(users, { fields: [boards.userId], references: [users.id] }),
  lists: many(lists),
}));

export const listsRelations = relations(lists, ({ one, many }) => ({
  board: one(boards, { fields: [lists.boardId], references: [boards.id] }),
  tasks: many(tasks),
}));

export const tasksRelations = relations(tasks, ({ one }) => ({
  list: one(lists, { fields: [tasks.listId], references: [lists.id] }),
}));

// Infer TypeScript Models directly from Database Schema
export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;

export type Board = typeof boards.$inferSelect;
export type NewBoard = typeof boards.$inferInsert;

export type List = typeof lists.$inferSelect;
export type NewList = typeof lists.$inferInsert;

export type Task = typeof tasks.$inferSelect;
export type NewTask = typeof tasks.$inferInsert;
```

---

### 3. Database Connection Pooling (`src/db/index.ts`)

```ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';
import * as dotenv from 'dotenv';
dotenv.config();

// Create connection pool (reusable TCP socket pool)
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                   // Maximum 20 simultaneous connections
  idleTimeoutMillis: 30000, // Close idle clients after 30 seconds
  connectionTimeoutMillis: 2000,
});

// Verify connection on startup
pool.on('connect', () => {
  console.log('✅ Connected to PostgreSQL Database Pool');
});

pool.on('error', (err) => {
  console.error('❌ Unexpected database error on idle client:', err);
  process.exit(-1);
});

// Export type-safe db client with complete relational schema loaded
export const db = drizzle(pool, { schema });
```

---

### 4. Running Migrations with Drizzle Kit

```bash
# 1. Generate SQL migration files from TypeScript schema
npx drizzle-kit generate

# 2. Apply migration files to the live PostgreSQL database
npx drizzle-kit migrate

# 3. (Optional) Open Drizzle Studio to inspect and edit live database rows
npx drizzle-kit studio
```

---

## Module 3: Type-Safe CRUD, Relational Queries & Transactions

### 1. Refactoring `BoardModel` with Drizzle ORM

```ts
// src/models/board.model.ts
import { db } from '../db';
import { boards, NewBoard, Board } from '../db/schema';
import { eq, desc } from 'drizzle-orm';

export const BoardModel = {
  findAllByUser: async (userId: string): Promise<Board[]> => {
    return db
      .select()
      .from(boards)
      .where(eq(boards.userId, userId))
      .orderBy(desc(boards.createdAt));
  },

  findById: async (id: string): Promise<Board | null> => {
    const result = await db
      .select()
      .from(boards)
      .where(eq(boards.id, id))
      .limit(1);

    return result[0] || null;
  },

  create: async (data: NewBoard): Promise<Board> => {
    const result = await db
      .insert(boards)
      .values(data)
      .returning(); // PostgreSQL returning clause gives back the inserted row!

    return result[0];
  },

  update: async (id: string, data: Partial<NewBoard>): Promise<Board | null> => {
    const result = await db
      .update(boards)
      .set({ ...data, updatedAt: new Date() })
      .where(eq(boards.id, id))
      .returning();

    return result[0] || null;
  },

  delete: async (id: string): Promise<boolean> => {
    const result = await db
      .delete(boards)
      .where(eq(boards.id, id))
      .returning({ deletedId: boards.id });

    return result.length > 0;
  },
};
```

---

### 2. Solving the N+1 Query Problem with Relational Queries

> [!IMPORTANT]
> **The N+1 Query Problem:** If a client requests a board with its lists and tasks, firing 1 query for the board, $N$ queries for lists, and $M$ queries for tasks destroys database performance.

Drizzle's **Relational Query API** loads the entire nested tree in an optimized, single database roundtrip:

```ts
// src/services/board.service.ts
import { db } from '../db';

export const BoardService = {
  getBoardWithFullDetails: async (boardId: string) => {
    // Eagerly fetch Board -> Lists -> Tasks in one query
    const boardWithHierarchy = await db.query.boards.findFirst({
      where: (boards, { eq }) => eq(boards.id, boardId),
      with: {
        lists: {
          orderBy: (lists, { asc }) => [asc(lists.position)],
          with: {
            tasks: {
              orderBy: (tasks, { asc }) => [asc(tasks.position)],
            },
          },
        },
      },
    });

    if (!boardWithHierarchy) {
      const error: any = new Error('Board not found');
      error.statusCode = 404;
      throw error;
    }

    return boardWithHierarchy;
  },
};
```

---

### 3. Under the Hood: ACID Database Transactions

When a user reorders multiple task cards or copies a template list, multiple database updates must succeed **atomically**. If step 3 fails, steps 1 and 2 must be **rolled back**:

```mermaid
sequenceDiagram
    autonumber
    participant App as Express Service
    participant DB as PostgreSQL Transaction Engine

    App->>DB: BEGIN TRANSACTION
    App->>DB: UPDATE tasks SET position = 0 WHERE id = 'task-A'
    App->>DB: UPDATE tasks SET position = 1 WHERE id = 'task-B'
    alt Network Glitch / Constraint Error on Task C
        App->>DB: ROLLBACK
        Note over DB: Database restores original state (Zero partial corruption!)
    else All Queries Succeed
        App->>DB: COMMIT
        Note over DB: Changes permanently written to disk
    end
```

```ts
// Example: Atomic List Position Reordering in Drizzle
export async function reorderLists(reorderedItems: { id: string; position: number }[]) {
  return db.transaction(async (tx) => {
    for (const item of reorderedItems) {
      await tx
        .update(lists)
        .set({ position: item.position })
        .where(eq(lists.id, item.id));
    }
  });
}
```

---

### Common Pitfalls & How to Avoid Them

| Pitfall | Root Cause | Fix |
| :--- | :--- | :--- |
| **Missing Foreign Key Cascades** | Deleting a board throws a foreign key constraint violation because child lists still exist. | Add `{ onDelete: 'cascade' }` to foreign key definitions in `schema.ts`. |
| **Connection Leak** | Creating a new `new Pool()` inside route handlers exhausts all database sockets. | Instantiate a single `Pool` in `src/db/index.ts` and export the shared `db` instance. |
| **Unindexed search queries** | Searching millions of tasks by title without an index causes full table scans ($O(n)$). | Add composite or B-tree indexes to foreign key and search columns. |

---

### Week 16: Comprehensive Assignment

**Objective:** Replace Flowboard's in-memory storage with a persistent PostgreSQL database managed via Drizzle ORM, featuring foreign key cascades, migration generation, and nested relational queries.

**Requirements:**
1. **Database Provisioning:**
   - PostgreSQL database provisioned on Neon.tech (or local Docker PostgreSQL).
   - `.env` configured with `DATABASE_URL` and excluded in `.gitignore`.
2. **Schema & Migrations:**
   - Complete schema in `src/db/schema.ts` for `users`, `boards`, `lists`, and `tasks` with UUID primary keys and `ON DELETE CASCADE`.
   - Migration generated and applied using `drizzle-kit generate` and `drizzle-kit migrate`.
3. **Persistent CRUD:**
   - `BoardModel` refactored to execute type-safe SQL queries via Drizzle ORM.
   - Tested that creating, updating, and deleting boards persists across server restarts.
4. **Relational Query:**
   - Implemented `GET /api/v1/boards/:id/full` returning the board along with all nested `lists` and their corresponding `tasks`.
5. **Git Workflow:**
   - Developed on branch `feature/postgres-drizzle-orm`, merged via Pull Request, and pulled locally.

---

## Recommended References & Deep-Dive Resources

### Official Documentation & Standards
- [PostgreSQL 16 Official Documentation](https://www.postgresql.org/docs/current/index.html) — The comprehensive manual for SQL syntax, data types, indexing, and transactional isolation levels.
- [Drizzle ORM Official Documentation](https://orm.drizzle.team/docs/overview) — Guide to schema declaration, migrations with Drizzle Kit, and relational queries.
- [Neon Serverless Postgres Documentation](https://neon.tech/docs/introduction) — Serverless database branching, autoscaling, and connection pooling.

### Recommended Textbooks & Canonical Books
- **Designing Data-Intensive Applications** by Martin Kleppmann (O'Reilly) — *Chapters 2, 3, & 7: Data Models and Query Languages, Storage and Retrieval (B-Trees), and Transactions (ACID).*
- **PostgreSQL: Up and Running (3rd Edition)** by Regina Obe and Leo Hsu (O'Reilly) — *Practical guide to administering, configuring, and optimizing PostgreSQL.*
- **SQL Antipatterns: Avoiding the Pitfalls of Database Programming** by Bill Karwin (Pragmatic Bookshelf) — *Classic relational design pitfalls including Jaywalking, Entity-Attribute-Value, and Naive Trees.*

### Deep Dives & Optimization Guides
- [Use The Index, Luke! A Guide to Database Performance](https://use-the-index-luke.com/) — Markus Winand's definitive deep dive into B-Tree indexing, execution plans, and join algorithms.
- [Drizzle ORM: Relational Queries Explained](https://orm.drizzle.team/docs/rqb) — Solving the N+1 problem with single-query join batching.
