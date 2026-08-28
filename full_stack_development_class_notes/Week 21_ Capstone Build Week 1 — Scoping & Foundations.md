# Week 21: Capstone Build Week 1 — Scoping & Foundations

All core software engineering disciplines — semantic HTML, CSS layout systems, modern TypeScript, React, Next.js App Router, Express REST APIs, PostgreSQL relational modeling, authentication security, Zod validation, automated testing, Docker containerization, and CI/CD pipelines — have been formally taught. In Weeks 21 and 22, you synthesize these tools into a single, cohesive, production-deployed **Full-Stack Capstone Project**.

This week focuses on **Product Scoping**, **Critical Path Modeling**, **Relational Schema Architecture**, and establishing the **Minimum Viable Product (MVP)** foundation.

---

## Module 1: Product Scoping & Engineering Tradeoffs

### 1. The Core Terminology of Technical Scoping

| Term | What It Actually Means | Technical Context |
| :--- | :--- | :--- |
| **Minimum Viable Product (MVP)** | The smallest possible slice of a product that delivers the core value proposition to end users end-to-end. | Prevents starting dozens of half-finished features; ensures one complete user workflow is built and hardened. |
| **Scope Creep** | The uncontrolled expansion of project requirements as development progresses, without adjustments to time or resources. | The primary reason software projects fail deadlines. Must be aggressively countered by pruning secondary features. |
| **Critical User Path (Happy Path)** | The sequential journey a user takes through an application to achieve their primary goal (e.g., Signup $\to$ Create Board $\to$ Add Task $\to$ Mark Complete). | Must function flawlessly before auxiliary features (avatars, dark mode, social logins) are built. |
| **Non-Functional Requirements (NFRs)** | System qualities such as security (AuthN/AuthZ), performance ($<200\text{ms}$ API response), reliability, and testability. | Architectural constraints that must be built into the foundation, not bolted on at the end. |

---

### 2. Choosing Your Capstone Path

```mermaid
flowchart TD
    Start([Choose Capstone Path]) --> Choice{Which path aligns with your engineering goals?}
    
    Choice -->|"Path A: Deep Polish & Feature Expansion"| Flowboard["<b>Path A: Production Flowboard (Kanban App)</b><br/>• Extend proven Week 15-20 architecture<br/>• Drag-and-drop task reordering<br/>• List management & member invitations<br/>• Low architectural risk, high depth & polish"]
    
    Choice -->|"Path B: Novel Domain Application"| Bespoke["<b>Path B: Bespoke Full-Stack Application</b><br/>• Pitch custom idea (Real Estate, Clinic Booking, Logistics)<br/>• Must use the EXACT same proven stack<br/>• High ownership, higher scoping discipline required"]
    
    Bespoke --> ScopeCheck{"Can the core MVP workflow be completed in 10 days?"}
    ScopeCheck -->|"No (Requires complex payments, websockets, AI models)"| Prune["Prune Secondary Scope Down to 1 Core Workflow"]
    Prune --> ScopeCheck
    ScopeCheck -->|"Yes (Realistic & Scoped)"| Approved["Ready for Schema Review"]
    Flowboard --> Approved
```

---

### 3. The Scope Downsizing Matrix: What to Keep vs. What to Cut

When building under tight engineering timelines, apply the **MoSCoW Prioritization Framework**:

| Priority Level | Description | Capstone Examples (Keep vs. Cut) |
| :--- | :--- | :--- |
| **Must Have (P0 - MVP)** | Essential to the core workflow; app fails without it. | • Secure Session Login / Signup<br>• Relational CRUD with foreign keys<br>• Server-side Zod validation<br>• Protected user data scoping (IDOR defense) |
| **Should Have (P1)** | High value, but app still functions without it. | • Real-time search filtering<br>• Optimistic UI state updates<br>• Error toast notifications |
| **Could Have (P2)** | Nice-to-have if time permits. | • User avatar uploads<br>• Due date calendar picker<br>• Export to CSV/JSON |
| **Won't Have (P3 - Cut!)** | Explicitly out of scope for Capstone MVP. | • Third-party OAuth (Google/GitHub)<br>• Live WebSocket multiplayer<br>• Real payment gateways (use mock flow instead) |

---

## Module 2: Data Modeling Workshop

### 1. From User Stories to Relational Entity Schemas

Before writing backend code, convert user stories into a formalized **Entity-Relationship Diagram (ERD)**:

```mermaid
erDiagram
    ORGANIZATIONS ||--o{ USERS : "1. belongs to"
    USERS ||--o{ PROJECTS : "2. creates / owns"
    PROJECTS ||--o{ ITEMS : "3. contains"
    ITEMS ||--o{ AUDIT_LOGS : "4. tracks"

    USERS {
        uuid id PK
        varchar email UK
        text password_hash
        timestamp created_at
    }

    PROJECTS {
        uuid id PK
        uuid user_id FK "ON DELETE CASCADE"
        varchar title
        text description
        timestamp created_at
    }

    ITEMS {
        uuid id PK
        uuid project_id FK "ON DELETE CASCADE"
        varchar title
        varchar status "enum: pending, active, completed"
        integer priority "0: Low, 1: Medium, 2: High"
        timestamp due_date
    }
```

---

### 2. Schema Architecture Review Checklist

Before executing migrations on your production database, verify these relational integrity rules:

1. **Foreign Key Deletion Rules:**
   - Does every `references(() => table.id)` have an explicit `{ onDelete: 'cascade' }` or `{ onDelete: 'restrict' }`?
   - *Why:* Prevents orphaned rows or accidental database deadlocks.
2. **Unique Constraints & Indexes:**
   - Are unique business keys (e.g., `email`, `slug`) marked with `.unique()`?
   - Are foreign key columns indexed for fast join performance?
3. **Data Type Precision:**
   - Are textual fields constrained with maximum lengths (e.g., `varchar('title', { length: 150 })`) rather than unbounded `text` where appropriate?
   - Are timestamps configured with `{ withTimezone: true }`?

---

## Module 3: Engineering Discipline & 1:1 Code Review

### 1. Daily Standup Cadence

Structure build sessions around a 10-minute standup answering three questions:
1. **What concrete milestone did you complete yesterday?** (e.g., "Implemented Zod validation on task creation endpoints").
2. **What single deliverable will you complete today?** (e.g., "Build the Next.js Server Component dashboard").
3. **What blockers or unknown technical risks are slowing you down?** (e.g., "CORS cookie rejection between Vercel and Railway").

---

### 2. Production Code Review Checklist

During 1:1 reviews with instructors and peers, every Pull Request is audited against these 5 pillars:

```mermaid
graph TD
    Audit["<b>Code Review Audit</b>"]
    Audit --> P1["<b>1. Validation (Edge):</b><br/>Is all incoming data validated via Zod?"]
    Audit --> P2["<b>2. Authorization (IDOR):</b><br/>Is every query scoped to req.user.id?"]
    Audit --> P3["<b>3. Error Flow:</b><br/>Do errors pass through AppError to global middleware?"]
    Audit --> P4["<b>4. Type Safety:</b><br/>Is code 100% strict TypeScript (Zero 'any')?"]
    Audit --> P5["<b>5. Test Coverage:</b><br/>Does a Supertest integration test prove the happy path?"]
```

---

### Week 21: Comprehensive Assignment

**Objective:** Lock in your capstone technical scope, receive instructor approval on your database ERD, generate Drizzle migrations, and build the end-to-end authenticated core happy path.

**Requirements:**
1. **1-Page Technical Specification Document:**
   - Problem statement and target user.
   - Core MVP workflow (step-by-step user path).
   - Explicit "Out of Scope" list.
   - Full ERD diagram sketch.
2. **Database Provisioning & Migrations:**
   - PostgreSQL schema declared in Drizzle ORM and successfully migrated to a live cloud database.
3. **Core Backend Endpoints:**
   - User authentication routes (`signup`, `login`, `logout`) operational with Bcrypt hashing and session cookies.
   - Core entity CRUD routes operational with Zod validation.
4. **Initial Next.js Frontend Wiring:**
   - Next.js application scaffolded with App Router and Tailwind CSS.
   - Authenticated dashboard page successfully reading data from backend via Server Components.
5. **Git Workflow:**
   - Work managed on feature branches (`feature/capstone-auth`, `feature/capstone-schema`), committed with conventional commits, and opened as PRs.
