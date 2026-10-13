# Week 17: Authentication, Authorization & Security

Welcome to Week 17! In Week 16, you connected Flowboard to a live PostgreSQL database. However, right now your API has zero security:
- Anyone in the world can delete anyone else's board simply by guessing an ID!
- There are no user accounts, passwords, or permissions.

In the real world, web applications must protect user data. When you log into Twitter or your bank:
1. The app verifies **who you are** (**Authentication**).
2. The app verifies **what you are allowed to see and touch** (**Authorization**).

This week, you master production-grade **Password Hashing with Bcrypt**, **JSON Web Tokens (JWT)**, **Authentication Middleware**, and **Authorization Guards** in Express.

---

## Pedagogical Roadmap

```mermaid
flowchart TD
    M1["<b>Module 1: Authentication vs. Authorization</b><br/>• AuthN (Passport) vs AuthZ (VIP Lounge Badge)<br/>• Why plain-text passwords are a crime<br/>• Hashing with Bcrypt (The Smoothie Mental Model)"]
    --> M2["<b>Module 2: How Tokens Work (JWTs)</b><br/>• The Concert Wristband Analogy<br/>• Anatomy of a JWT: Header, Payload, Signature<br/>• Signing and verifying tokens"]
    --> M3["<b>Module 3: Building User Registration & Login</b><br/>• POST /api/auth/register (Hashing & saving)<br/>• POST /api/auth/login (Comparing hashes & issuing token)<br/>• Storing JWTs safely in the client"]
    --> M4["<b>Module 4: Protecting Routes with Middleware</b><br/>• The Bouncer Middleware: authenticateToken<br/>• Attaching req.user to incoming requests<br/>• Returning 401 Unauthorized vs 403 Forbidden"]
    --> M5["<b>Module 5: Authorization & Preventing IDOR</b><br/>• Insecure Direct Object References explained<br/>• Enforcing user ownership in SQL queries<br/>• Users only see their own boards!"]
```

---

## Module 1: Authentication vs. Authorization & Password Hashing

### 1. AuthN vs. AuthZ (The Airport Mental Model)

> [!NOTE]
> **The Mental Model:**
> - **Authentication (AuthN):** Showing your **International Passport** at airport immigration. It answers: *"Are you truly who you claim to be?"*
> - **Authorization (AuthZ):** Showing your **First-Class Boarding Pass** at the VIP lounge door. It answers: *"Do you have permission to enter this room and drink the free champagne?"*

---

### 2. Password Hashing: The Fruit Smoothie Mental Model

> [!CAUTION]
> **The #1 Rule of Backend Engineering:** **NEVER, EVER store passwords in plain text!**
> If your database is ever leaked or hacked, plain-text passwords expose every single user to identity theft.

How do we verify a user's password without saving their real password? We use a **One-Way Cryptographic Hash** called **Bcrypt**:

> [!TIP]
> **The Mental Model:** Think of Bcrypt as a **blender**.
> - You put an apple and a mango into the blender $\to$ you get a specific smoothie.
> - Anyone with the same recipe gets the exact same smoothie.
> - But **nobody can un-blend a smoothie back into an apple!**
> 
> When a user logs in, you put their submitted password into the blender and compare the smoothies. If the smoothies match, the password was correct!

```mermaid
flowchart LR
    PWD["Plain Password: 'mypassword123'"] --> BCRYPT["Bcrypt Blender + Salt"] --> HASH["Stored Hash: $2b$10$e8uq..."]
    HASH -.->|"CANNOT BE REVERSED!"| PWD
```

#### Installing Bcrypt & Types:
```bash
npm install bcryptjs jsonwebtoken
npm install -D @types/bcryptjs @types/jsonwebtoken
```

#### Hashing and Comparing Passwords in Code:
```ts
import bcrypt from "bcryptjs";

// 1. When user registers: Hash the password (cost factor = 10 rounds):
const plainPassword = "superSecretPassword123";
const passwordHash = await bcrypt.hash(plainPassword, 10);
console.log("Stored in Database:", passwordHash);
// Output looks like: $2b$10$wKq...

// 2. When user logs in: Compare candidate password against stored hash:
const isMatch = await bcrypt.compare("superSecretPassword123", passwordHash);
console.log("Password correct?", isMatch); // true

const isWrong = await bcrypt.compare("wrongGuess", passwordHash);
console.log("Password correct?", isWrong); // false
```

---

## Module 2: How Tokens Work (JSON Web Tokens - JWT)

### 1. The Concert Wristband Analogy

> [!NOTE]
> **The Mental Model:** When you buy a ticket at an event venue, the cashier checks your ID once and clips a **stamped, tamper-proof wristband** onto your arm.
> For the rest of the night, whenever you visit the bar, food stands, or stage, the security guard doesn't call the bank or check your passport again — they just look at your stamped wristband!
> A **JWT (JSON Web Token)** is an electronic, cryptographically signed wristband for the web.

```mermaid
sequenceDiagram
    actor User
    participant Server as Express Server
    participant DB as PostgreSQL

    User->>Server: POST /api/auth/login { email, password }
    Server->>DB: Check email & compare hash
    Server-->>User: Returns JWT Wristband string! (Bearer eyJhbGci...)

    Note over User,Server: Subsequent Protected Request
    User->>Server: GET /api/boards (Header: Authorization: Bearer eyJhbGci...)
    Note over Server: Server verifies signature with secret key
    Server-->>User: 200 OK with User's boards!
```

---

### 2. Anatomy of a JWT

A JWT looks like a long string of letters separated by two dots:
`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIxMjM0In0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c`

1. **Header (Red):** Specifies algorithm (e.g., `HS256`).
2. **Payload (Purple):** The data stored inside (e.g., `{ userId: "123", email: "user@lagos.com" }`).
3. **Signature (Blue):** A mathematical seal signed with your secret server key (`JWT_SECRET`). If an attacker changes a single letter in the payload, the signature breaks immediately!

---

## Module 3: Building User Registration & Login in Flowboard

### 1. The Users Database Table (`src/db/schema.ts`)

```ts
import { pgTable, text, timestamp, uuid, varchar } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").defaultRandom().primaryKey(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  passwordHash: text("password_hash").notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

// Update boards table to reference users:
export const boards = pgTable("boards", {
  id: uuid("id").defaultRandom().primaryKey(),
  userId: uuid("user_id")
    .notNull()
    .references(() => users.id, { onDelete: "cascade" }), // Delete user's boards if user is deleted
  title: varchar("title", { length: 255 }).notNull(),
  description: text("description").default(""),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```
Run `npx drizzle-kit push` to apply the updated schema to Neon!

---

### 2. Register & Login Endpoints

```ts
// src/routes/auth.routes.ts
import { Router, Request, Response } from "express";
import bcrypt from "bcryptjs";
import jwt from "jsonwebtoken";
import { eq } from "drizzle-orm";
import { db } from "../db";
import { users } from "../db/schema";

export const authRouter = Router();
const JWT_SECRET = process.env.JWT_SECRET || "super-secret-lagos-key";

// 1. POST /api/auth/register
authRouter.post("/register", async (req: Request, res: Response) => {
  const { email, password } = req.body;

  if (!email || !password || password.length < 6) {
    return res.status(400).json({ error: "Password must be at least 6 characters long." });
  }

  try {
    // Check if email already registered:
    const [existing] = await db.select().from(users).where(eq(users.email, email.toLowerCase()));
    if (existing) {
      return res.status(409).json({ error: "Email is already registered." });
    }

    // Hash password with 10 salt rounds:
    const passwordHash = await bcrypt.hash(password, 10);

    // Save user to database:
    const [newUser] = await db
      .insert(users)
      .values({ email: email.toLowerCase(), passwordHash })
      .returning({ id: users.id, email: users.email });

    res.status(201).json({ message: "Registration successful!", user: newUser });
  } catch (error) {
    res.status(500).json({ error: "Registration failed." });
  }
});

// 2. POST /api/auth/login
authRouter.post("/login", async (req: Request, res: Response) => {
  const { email, password } = req.body;

  try {
    const [user] = await db.select().from(users).where(eq(users.email, email.toLowerCase()));
    if (!user) {
      return res.status(401).json({ error: "Invalid email or password." });
    }

    // Verify password:
    const isValid = await bcrypt.compare(password, user.passwordHash);
    if (!isValid) {
      return res.status(401).json({ error: "Invalid email or password." });
    }

    // Sign a JWT valid for 24 hours:
    const token = jwt.sign(
      { userId: user.id, email: user.email },
      JWT_SECRET,
      { expiresIn: "24h" }
    );

    res.json({ message: "Login successful!", token });
  } catch (error) {
    res.status(500).json({ error: "Login failed." });
  }
});
```

---

## Module 4: The Bouncer Middleware (`authenticateToken`)

We need a middleware function that acts like a nightclub bouncer:
- Checks the `Authorization: Bearer <token>` header.
- If missing or invalid $\to$ halts request and returns `401 Unauthorized`.
- If valid $\to$ decodes user information, attaches `req.user`, and calls `next()`!

```ts
// src/middleware/auth.middleware.ts
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";

const JWT_SECRET = process.env.JWT_SECRET || "super-secret-lagos-key";

// Extend Express Request type to include user:
export interface AuthRequest extends Request {
  user?: { userId: string; email: string };
}

export function authenticateToken(req: AuthRequest, res: Response, next: NextFunction) {
  const authHeader = req.headers["authorization"];
  const token = authHeader && authHeader.split(" ")[1]; // Format: "Bearer <token>"

  if (!token) {
    return res.status(401).json({ error: "Access denied. No token provided." });
  }

  try {
    const decoded = jwt.verify(token, JWT_SECRET) as { userId: string; email: string };
    req.user = decoded; // Attach authenticated user to request object!
    next(); // Pass to next handler
  } catch (err) {
    return res.status(403).json({ error: "Invalid or expired token." });
  }
}
```

---

## Module 5: Authorization & Preventing IDOR

> [!CAUTION]
> **What is IDOR (Insecure Direct Object Reference)?**
> If user Alice requests `GET /api/boards/123`, and your server just does:
> ```ts
> db.select().from(boards).where(eq(boards.id, req.params.id));
> ```
> An attacker can simply type `GET /api/boards/124`, `125`, `126` and view everyone's private boards!
> 
> **The Golden Rule:** Always filter queries by **BOTH** the item ID and the authenticated user's ID:

```ts
// src/routes/board.routes.ts
import { Router, Response } from "express";
import { and, eq } from "drizzle-orm";
import { authenticateToken, AuthRequest } from "../middleware/auth.middleware";
import { db } from "../db";
import { boards } from "../db/schema";

export const boardRouter = Router();

// Apply auth middleware to ALL routes in this router:
boardRouter.use(authenticateToken);

// 1. GET /api/boards - Returns ONLY boards belonging to the logged-in user:
boardRouter.get("/", async (req: AuthRequest, res: Response) => {
  const userId = req.user!.userId;
  const userBoards = await db.select().from(boards).where(eq(boards.userId, userId));
  res.json(userBoards);
});

// 2. POST /api/boards - Automatically attaches the logged-in user's ID:
boardRouter.post("/", async (req: AuthRequest, res: Response) => {
  const { title, description } = req.body;
  const userId = req.user!.userId;

  const [createdBoard] = await db
    .insert(boards)
    .values({
      userId,
      title,
      description: description || "",
    })
    .returning();

  res.status(201).json(createdBoard);
});

// 3. DELETE /api/boards/:id - Prevents deleting another user's board!
boardRouter.delete("/:id", async (req: AuthRequest, res: Response) => {
  const boardId = req.params.id;
  const userId = req.user!.userId;

  // Enforce ownership: Must match both boardId AND userId!
  const deleted = await db
    .delete(boards)
    .where(and(eq(boards.id, boardId), eq(boards.userId, userId)))
    .returning();

  if (deleted.length === 0) {
    return res.status(404).json({ error: "Board not found or you lack permission to delete it." });
  }

  res.json({ message: "Board deleted successfully." });
});
```

---

## Module 6: Rookie Traps & Debugging Guide

| Error Message / Trap | Root Cause | How to Fix It |
| :--- | :--- | :--- |
| `jwt must be provided` | Missing token in request header. | In Thunder Client/Postman, go to **Auth** $\to$ choose **Bearer Token** and paste your JWT. |
| `jwt malformed` | Passed an invalid string or forgot `"Bearer "` prefix. | Ensure format is `Authorization: Bearer eyJhbGci...`. |
| Passwords matching fails on login | You re-hashed the incoming password instead of using `bcrypt.compare`. | Always use `await bcrypt.compare(submittedPassword, storedHash)`. |
| Storing JWT in plain `localStorage` in production | Vulnerable to XSS token theft. | For sensitive financial apps, use `HttpOnly` cookies. For standard SPA APIs, store securely in memory or auth context. |

---

## 🎯 Week 17 Hands-on Classroom Exercises

### Level 1: Registration & Hashing
1. Implement the `POST /api/auth/register` endpoint.
2. In Thunder Client, register a new user: `{ "email": "student@lagosdataschool.com", "password": "password123" }`.
3. Open your Neon database console and inspect the `users` table: verify that the password is a cryptographically hashed string!

### Level 2: Login & Generating JWTs
1. Implement `POST /api/auth/login`.
2. Login with the correct password and receive your JWT token.
3. Test logging in with the wrong password and confirm you receive `401 Unauthorized`.

### Level 3: Securing Flowboard with Bouncer Middleware
1. Protect all board routes with `authenticateToken`.
2. Register User A and User B.
3. Have User A create a board.
4. Try fetching all boards with User B's token: verify that User B **cannot see** User A's board!

---

## 📚 Curated Deep-Dive References

* [The Valley of Code: Authentication](https://thevalleyofcode.com/auth/) — Flavio Copes' beginner guide to web authentication.
* [JWT.io: Introduction to JSON Web Tokens](https://jwt.io/introduction) — Visual debugger and interactive specification for JWTs.
* [OWASP Top 10 Security Risks](https://owasp.org/www-project-top-ten/) — The industry authority on securing web applications against injection and broken access controls.
