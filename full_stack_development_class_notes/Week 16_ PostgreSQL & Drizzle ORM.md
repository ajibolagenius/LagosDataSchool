# Week 16: PostgreSQL & Drizzle ORM

Welcome to Week 16! In Week 15, you built your first backend server with Node.js and Express. It worked, but it had one massive flaw:
**Every time the server restarted, all created boards vanished!**

In-memory arrays only live in your computer's RAM. To build real production software, you need a **Database** — a dedicated, high-performance storage engine that writes data permanently to disk and allows fast searching across millions of records.

This week, you master **PostgreSQL** (the world's most trusted relational database) and **Drizzle ORM** (a modern, lightning-fast TypeScript tool for querying databases).

---

## Pedagogical Roadmap

```mermaid
flowchart TD
    M1["<b>Module 1: Relational Databases (The Filing Cabinet Analogy)</b><br/>• Why not store data in JSON files?<br/>• Tables, Rows, and Columns<br/>• Primary Keys (ID) and Foreign Keys (Relationships)"]
    --> M2["<b>Module 2: Free Cloud Database Setup (Neon)</b><br/>• Provisioning a free serverless PostgreSQL database<br/>• Understanding the DATABASE_URL connection string<br/>• Securing secrets in .env"]
    --> M3["<b>Module 3: What is an ORM? & Intro to Drizzle</b><br/>• Why write TypeScript instead of raw SQL?<br/>• Defining tables in schema.ts<br/>• Pushing tables to the database with drizzle-kit push"]
    --> M4["<b>Module 4: Type-Safe CRUD with Drizzle</b><br/>• db.select().from(boards)<br/>• db.insert(boards).values(...)<br/>• Filtering with eq() & and()<br/>• db.update() and db.delete()"]
    --> M5["<b>Module 5: Replacing Flowboard In-Memory Arrays</b><br/>• Connecting Express routes directly to PostgreSQL<br/>• Permanent, bulletproof data persistence!"]
```

---

## Module 1: Relational Databases (The Filing Cabinet Analogy)

### 1. Why Not Just Save Data in a `data.json` File?

Beginners often ask: *"Why can't I just write data to a JSON file using `fs.writeFile`?"*

Imagine 100 users clicking "Submit" at the exact same fraction of a second:
- Two users trying to write to the same text file simultaneously will corrupt the file.
- Finding one specific task out of 500,000 in a text file requires reading the entire giant file into memory, crashing your server.

> [!NOTE]
> **The Mental Model:** A **Relational Database (PostgreSQL)** is like an automated, indestructible, high-security **Filing Cabinet System**:
> - Each drawer is a **Table** (e.g., `boards`, `users`, `tasks`).
> - Inside each drawer, every folder is a **Row** (a single board or user).
> - Every folder has pre-printed form fields called **Columns** (e.g., `title`, `created_at`).
> - It can handle thousands of reads and writes per second without breaking a sweat!

---

### 2. Primary Keys & Foreign Keys (Connecting Data)

In a relational database, tables connect to each other through keys:

```mermaid
erDiagram
    USERS ||--o{ BOARDS : "owns (1:N)"
    BOARDS ||--o{ LISTS : "contains (1:N)"
    LISTS ||--o{ TASKS : "contains (1:N)"

    USERS {
        uuid id PK "Primary Key (Unique Badge)"
        string email
    }
    BOARDS {
        uuid id PK
        uuid user_id FK "Foreign Key -> References users(id)"
        string title
    }
    LISTS {
        uuid id PK
        uuid board_id FK "Foreign Key -> References boards(id)"
        string title
    }
    TASKS {
        uuid id PK
        uuid list_id FK "Foreign Key -> References lists(id)"
        string title
    }
```

- **Primary Key (PK):** A unique identifier for every row (like an ID card number or National Identity Number). No two rows can ever have the same Primary Key.
- **Foreign Key (FK):** A column that points to the Primary Key of another table. For example, a `board` has a `user_id` column telling us which user owns it.

---

## Module 2: Free Cloud Database Setup (Neon PostgreSQL)

Instead of struggling to install, configure, and run a heavy PostgreSQL database on your laptop, we will use **Neon** (`neon.tech`): a free, generous, cloud-hosted serverless PostgreSQL platform.

### Step-by-Step Neon Setup:
1. Go to [neon.tech](https://neon.tech) and sign up for a free account with GitHub.
2. Click **Create Project** $\to$ Name it `flowboard-db`.
3. Neon will display your **Connection String (DATABASE_URL)**:
   ```
   postgresql://username:password@ep-flowboard-123.eu-central-1.aws.neon.tech/flowboard?sslmode=require
   ```
4. Copy this string!

### Storing in `.env`:
Never put passwords or database URLs in your public GitHub repository! Create a `.env` file in your server root:

```env
PORT=5000
DATABASE_URL=postgresql://username:password@ep-flowboard-123.eu-central-1.aws.neon.tech/flowboard?sslmode=require
```

Make sure `.env` is listed inside your `.gitignore`!

---

## Module 3: What is an ORM? & Defining Schemas with Drizzle

### 1. What is an ORM?

> [!NOTE]
> **The Mental Model:** An **ORM (Object-Relational Mapper)** is a universal translator between TypeScript and SQL.
> Instead of writing messy, error-prone SQL strings by hand:
> ```sql
> SELECT * FROM "boards" WHERE "user_id" = '123';
> ```
> Drizzle lets you write clean, fully autocompleted TypeScript:
> ```ts
> await db.select().from(boards).where(eq(boards.userId, '123'));
> ```
> If you make a typo in a column name, TypeScript catches it before you run the code!

---

### 2. Installing Drizzle ORM

```bash
# 1. Install Drizzle ORM and Neon Postgres driver
npm install drizzle-orm @neondatabase/serverless dotenv

# 2. Install developer migration tooling
npm install -D drizzle-kit
```

Create `drizzle.config.ts`:
```ts
// drizzle.config.ts
import { defineConfig } from "drizzle-kit";
import * as dotenv from "dotenv";
dotenv.config();

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

---

### 3. Defining the Flowboard Database Schema (`src/db/schema.ts`)

```ts
// src/db/schema.ts
import { pgTable, text, timestamp, uuid, varchar } from "drizzle-orm/pg-core";

// 1. Boards Table:
export const boards = pgTable("boards", {
  id: uuid("id").defaultRandom().primaryKey(),
  title: varchar("title", { length: 255 }).notNull(),
  description: text("description").default(""),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

// 2. Lists Table (belongs to a board):
export const lists = pgTable("lists", {
  id: uuid("id").defaultRandom().primaryKey(),
  boardId: uuid("board_id")
    .notNull()
    .references(() => boards.id, { onDelete: "cascade" }), // If board is deleted, delete its lists!
  title: varchar("title", { length: 255 }).notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

---

### 4. Pushing Tables to PostgreSQL with `drizzle-kit push`

Whenever you create or change a table in `schema.ts`, you push the structure to your live database with one command:

```bash
npx drizzle-kit push
```

Drizzle connects to Neon, creates the tables, sets up the primary keys, and confirms: `[✓] Changes applied!`

---

## Module 4: Type-Safe CRUD with Drizzle

Create your database connection instance in `src/db/index.ts`:

```ts
// src/db/index.ts
import { neon } from "@neondatabase/serverless";
import { drizzle } from "drizzle-orm/neon-http";
import * as schema from "./schema";
import * as dotenv from "dotenv";
dotenv.config();

const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql, { schema });
```

### The 4 Essential Drizzle Operations:

#### 1. Read All Items (`SELECT * FROM boards`):
```ts
import { db } from "./db";
import { boards } from "./db/schema";

const allBoards = await db.select().from(boards);
```

#### 2. Insert an Item (`INSERT INTO boards ... RETURNING *`):
```ts
const [newBoard] = await db
  .insert(boards)
  .values({
    title: "Launch Marketing Plan",
    description: "Prepare billboards and digital ads",
  })
  .returning(); // Returns the created row with its generated UUID!
```

#### 3. Find by ID with `where(eq(...))`:
```ts
import { eq } from "drizzle-orm";

const [board] = await db
  .select()
  .from(boards)
  .where(eq(boards.id, "some-uuid-here"));
```

#### 4. Update and Delete:
```ts
// Update title:
await db
  .update(boards)
  .set({ title: "Updated Title" })
  .where(eq(boards.id, "some-uuid-here"));

// Delete row:
await db
  .delete(boards)
  .where(eq(boards.id, "some-uuid-here"));
```

---

## Module 5: Wiring Express Routes Directly to PostgreSQL

Now replace the in-memory array in your `src/index.ts` with real database queries:

```ts
// src/index.ts
import express, { Request, Response } from "express";
import cors from "cors";
import { eq } from "drizzle-orm";
import { db } from "./db";
import { boards } from "./db/schema";

const app = express();
app.use(cors());
app.use(express.json());

// 1. GET /api/boards - List all boards from Postgres
app.get("/api/boards", async (req: Request, res: Response) => {
  try {
    const data = await db.select().from(boards);
    res.json(data);
  } catch (err) {
    res.status(500).json({ error: "Failed to fetch boards." });
  }
});

// 2. POST /api/boards - Save new board to Postgres
app.post("/api/boards", async (req: Request, res: Response) => {
  const { title, description } = req.body;
  if (!title || title.trim().length < 3) {
    return res.status(400).json({ error: "Title must be at least 3 characters." });
  }

  try {
    const [created] = await db
      .insert(boards)
      .values({
        title: title.trim(),
        description: description ? description.trim() : "",
      })
      .returning();

    res.status(201).json(created);
  } catch (err) {
    res.status(500).json({ error: "Failed to save board to database." });
  }
});

// 3. GET /api/boards/:id - Find board by ID
app.get("/api/boards/:id", async (req: Request, res: Response) => {
  try {
    const [board] = await db.select().from(boards).where(eq(boards.id, req.params.id));
    if (!board) return res.status(404).json({ error: "Board not found" });
    res.json(board);
  } catch (err) {
    res.status(500).json({ error: "Database error." });
  }
});

app.listen(5000, () => {
  console.log("Flowboard API connected to PostgreSQL on http://localhost:5000");
});
```

Now, when you restart your server or close your laptop, **your data remains safely preserved in PostgreSQL forever!**

---

## Module 6: Rookie Traps & Debugging Guide

| Error Message / Trap | Root Cause | How to Fix It |
| :--- | :--- | :--- |
| `DATABASE_URL is not defined` | Missing `.env` file or forgot `dotenv.config()`. | Check that `.env` has `DATABASE_URL=...` and import `dotenv` at the top of entry files. |
| `invalid input syntax for type uuid` | You tried querying a UUID column with a non-UUID string like `"1"`. | Ensure you are passing real UUID strings generated by Postgres (`defaultRandom()`). |
| `drizzle-kit push` fails with authentication error | Password or username in connection string is incorrect or expired. | Copy a fresh connection string from your Neon dashboard. |
| Forgetting `.returning()` on insert | Drizzle executes the insert, but `result` returns undefined instead of the created item. | Always chain `.returning()` if you need the created record back immediately. |

---

## 🎯 Week 16 Hands-on Classroom Exercises

### Level 1: Neon Connection & Schema Push
1. Create a free PostgreSQL database on Neon.
2. Configure `.env` and `drizzle.config.ts`.
3. Declare the `boards` table in `src/db/schema.ts` and run `npx drizzle-kit push`.
4. Open the Neon Web Console and inspect your newly created live SQL table!

### Level 2: Real Database CRUD in Express
1. Connect `GET /api/boards` and `POST /api/boards` to Drizzle.
2. In Thunder Client, create 3 real boards.
3. Stop and restart your Express server terminal.
4. Run `GET /api/boards` and confirm that all 3 boards are still there!

### Level 3: Adding the Lists Table
1. Define a `lists` table in `schema.ts` with a foreign key referencing `boards.id`.
2. Push the schema to Neon.
3. Build `POST /api/boards/:boardId/lists` to insert a list for a specific board!

---

## 📚 Curated Deep-Dive References

* [The Valley of Code: SQL Tutorial](https://thevalleyofcode.com/sql/) — Flavio Copes' beginner guide to relational databases.
* [Drizzle ORM Official Documentation](https://orm.drizzle.team/docs/overview) — The premier guide to Drizzle schema definitions and queries.
* [Neon PostgreSQL Documentation](https://neon.tech/docs/introduction) — Serverless PostgreSQL architecture and features.
* [PostgreSQL Official Documentation](https://www.postgresql.org/docs/) — The authoritative reference for PostgreSQL data types and constraints.
