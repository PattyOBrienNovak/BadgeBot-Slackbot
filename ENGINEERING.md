# BadgBot — Engineering Spec

**Source:** BadgeBot PRD v0  
**Language:** TypeScript  
**Runtime:** Node.js 20+  
**Framework:** @slack/bolt (Socket Mode)  
**Database:** PostgreSQL (via `pg` + raw SQL or `kysely` query builder)

---

## Architecture Overview

```
┌──────────────┐     Socket Mode      ┌─────────────────┐
│  Slack API   │ ◄──────────────────► │   BadgBot App   │
└──────────────┘                      │   (app.ts)      │
                                      ├─────────────────┤
                                      │  handlers/      │
                                      │  services/      │
                                      │  db/            │
                                      └────────┬────────┘
                                               │
                                        ┌──────▼──────┐
                                        │  PostgreSQL  │
                                        └─────────────┘
```

---

## Environment Variables

All secrets are stored in `.env` on the local machine. This file is in `.gitignore` and must never be committed.

```
# .env
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
DATABASE_URL=postgresql://user:password@localhost:5432/badgbot
```

Use `.env.example` (committed, no real values) as the reference template.

---

## Project Structure

```
badgbot/
├── src/
│   ├── app.ts                  # Entry point — initializes Bolt and starts app
│   ├── db/
│   │   ├── client.ts           # pg Pool singleton
│   │   └── migrations/         # SQL migration files (numbered)
│   ├── handlers/
│   │   ├── lessonDelivery.ts   # Scheduled lesson message sender
│   │   ├── reflectionModal.ts  # Button action → open modal
│   │   ├── reflectionSubmit.ts # Modal submission handler
│   │   └── optOut.ts           # Opt-out command/button handler
│   ├── services/
│   │   ├── badgeService.ts     # Completion check + badge issuance
│   │   ├── reflectionService.ts# Save reflection, post to thread
│   │   └── lessonService.ts    # Query due lessons
│   └── types.ts                # Shared TypeScript interfaces
├── .env                        # Local secrets — NOT committed
├── .env.example                # Committed placeholder
├── .gitignore
├── package.json
├── tsconfig.json
└── ENGINEERING.md
```

---

## TypeScript Interfaces (`src/types.ts`)

```ts
export interface User {
  id: string;
  slack_id: string;
  email: string;
  display_name: string;
  opted_out: boolean;
  created_at: Date;
}

export interface Module {
  id: string;
  name: string;
  description: string;
  slack_channel_id: string;
  total_lessons: number;
  created_at: Date;
}

export interface Lesson {
  id: string;
  module_id: string;
  lesson_number: number;
  content_blocks: object[];
  materials: object[];
  scheduled_for: Date;
  daily_message_id: string | null;
  daily_message_ts: string | null;
  created_at: Date;
}

export interface Enrollment {
  id: string;
  user_id: string;
  module_id: string;
  certificate_id: string | null;
  issued_at: Date | null;
}

export interface Badge {
  id: string;
  user_id: string;
  enrollment_id: string;
  issued_at: Date;
  awaiting_sent: boolean;
}

export interface TheorySubmission {
  id: string;
  user_id: string;
  lesson_id: string;
  module_id: string;
  reflection_text: string;
  is_public: boolean;
  thread_message_id: string | null;
  submitted_at: Date;
}

export interface ReflectionThread {
  id: string;
  module_id: string;
  parent_message_id: string;
  created_at: Date;
}
```

---

## Build Plan — Milestones

Each milestone is independently testable before moving to the next.

---

### Milestone 1 — TypeScript Project Setup

**Goal:** Compile and run a typed Bolt app.

**Steps:**
1. Convert `app.js` → `app.ts`
2. Add `tsconfig.json` (target ES2020, module CommonJS, strict true, outDir `dist/`)
3. Install dev dependencies: `typescript`, `ts-node`, `@types/node`
4. Update `package.json` scripts:
   - `"dev": "ts-node src/app.ts"`
   - `"build": "tsc"`
   - `"start": "node dist/app.js"`

**Test:** `npm run dev` prints `BadgBot is running` with no TypeScript errors.

---

### Milestone 2 — Database Connection

**Goal:** Connect to PostgreSQL and confirm the connection at startup.

**Steps:**
1. Install `pg` and `@types/pg`
2. Create `src/db/client.ts` — exports a `pg.Pool` initialized from `DATABASE_URL`
3. In `app.ts`, call `pool.query('SELECT 1')` on startup and log success
4. Add `DATABASE_URL` to `.env` and `.env.example`

**Test:** App starts, logs `BadgBot is running` and `Database connected`.  
Intentionally break `DATABASE_URL` and confirm the app logs the error and exits.

---

### Milestone 3 — Database Migrations

**Goal:** Define and apply the full schema.

**Steps:**
1. Create `src/db/migrations/` with numbered SQL files:
   - `001_create_users.sql`
   - `002_create_modules.sql`
   - `003_create_lessons.sql`
   - `004_create_enrollments.sql`
   - `005_create_module_enrollments.sql`
   - `006_create_badges.sql`
   - `007_create_theory_submissions.sql`
   - `008_create_reflection_threads.sql`
2. Create a `src/db/migrate.ts` script that runs all pending migrations in order
3. Add `"migrate": "ts-node src/db/migrate.ts"` to scripts

**Test:** Run `npm run migrate`. Verify all tables exist in the database via `psql` or a DB client.  
Run again — confirm it's idempotent (no errors, no duplicate tables).

---

### Milestone 4 — Seed Data

**Goal:** Populate the database with a test module, lessons, a user, and an enrollment so all later milestones have real data to work with.

**Steps:**
1. Create `src/db/seed.ts` with:
   - 1 module (e.g. "Intro to Design Thinking", 3 lessons, a real Slack channel ID)
   - 3 lessons with `scheduled_for` set to testable times (one in the past, two in the future)
   - 1 user (your own Slack user ID)
   - 1 enrollment linking user to module
2. Add `"seed": "ts-node src/db/seed.ts"` to scripts

**Test:** Run `npm run seed`. Query the DB and confirm all rows exist. Run `npm run dev` — app still starts cleanly.

---

### Milestone 5 — Lesson Delivery

**Goal:** BadgBot sends a formatted Slack message for each due lesson.

**Steps:**
1. Create `src/services/lessonService.ts`:
   - `getDueLessons()` — queries lessons where `scheduled_for <= NOW()` and `daily_message_id IS NULL`, joined to `module_enrollments` for enrolled users
2. Create `src/handlers/lessonDelivery.ts`:
   - `deliverDueLessons(app)` — calls `getDueLessons()`, formats a Slack Block Kit message per lesson, posts to the user's DM or module channel, saves `daily_message_id` back to the lesson row
3. Call `deliverDueLessons` on a polling interval (every 60s) in `app.ts`

**Message format (Block Kit):**
```
[Header] Lesson {number}: {module name}
[Section] {content_blocks rendered as text}
[Section] Materials: {links if present}
[Actions] [Submit Reflection] button  |  [Opt Out] button
```

**Test:** Set a lesson's `scheduled_for` to now in the DB. Within 60s, confirm a Slack DM arrives with the correct content and the button. Confirm `daily_message_id` is written back to the DB. Confirm re-running the poller does NOT re-send the message.

---

### Milestone 6 — Reflection Modal

**Goal:** Clicking "Submit Reflection" opens a Slack modal.

**Steps:**
1. Create `src/handlers/reflectionModal.ts`:
   - Listen for `action_id: 'submit_reflection'`
   - Open a modal via `client.views.open` with:
     - Plain text input (reflection text, required, min length 1)
     - Static select: "Share publicly" / "Keep private"
     - Hidden `lesson_id` in `private_metadata`
2. Register the action handler in `app.ts`

**Test:** Click the button in Slack. Confirm the modal opens with both fields. Confirm submitting an empty reflection shows Slack's built-in validation error (required field).

---

### Milestone 7 — Reflection Submission

**Goal:** Modal submission saves the reflection and optionally posts it publicly.

**Steps:**
1. Create `src/services/reflectionService.ts`:
   - `saveReflection(data)` — inserts into `theory_submissions`
   - `postToReflectionThread(app, submission, module)` — posts to the module's reflection thread; creates the thread anchor in `reflection_threads` if it doesn't exist yet; saves `thread_message_id` on the submission
2. Create `src/handlers/reflectionSubmit.ts`:
   - Listen for `callback_id: 'reflection_modal'`
   - Parse values and `private_metadata`
   - Call `saveReflection`, then `postToReflectionThread` if `is_public`
   - Trigger `badgeService.checkAndIssueBadge` (Milestone 8)
3. Register the view submission handler in `app.ts`

**Test:**
- Submit a private reflection → row appears in `theory_submissions`, `is_public = false`, no thread post.
- Submit a public reflection → row appears, a message is posted in the module channel thread, `thread_message_id` is saved.
- Submit duplicate for same lesson → confirm the app handles gracefully (upsert or error log, no crash).

---

### Milestone 8 — Badge Issuance

**Goal:** When all lessons are complete, issue a badge and send a congratulations message.

**Steps:**
1. Create `src/services/badgeService.ts`:
   - `checkAndIssueBadge(userId, moduleId, app)`:
     1. Count rows in `theory_submissions` for this user + module
     2. Compare to `modules.total_lessons`
     3. If equal: insert into `badges` (`awaiting_sent = true`), update `enrollments` (`certificate_id`, `issued_at`), send congrats DM, set `awaiting_sent = false`
2. Congrats message format:
```
[Header] You did it! 🎉
[Section] You've completed {module name}.
[Section] Your badge has been issued. Certificate ID: {certificate_id}
[Image] Badge image (placeholder URL for v1)
```

**Test:**
- Manually insert reflections for all-but-one lesson. Submit the final one via Slack. Confirm congrats DM arrives, badge row exists with `awaiting_sent = false`, enrollment has `issued_at` set.
- Simulate a Slack API failure (bad token) on the congrats send — confirm `awaiting_sent` stays `true` and the badge row exists for retry.

---

### Milestone 9 — Opt-Out

**Goal:** Users can stop receiving messages.

**Steps:**
1. Create `src/handlers/optOut.ts`:
   - Listen for `action_id: 'opt_out'`
   - Set `users.opted_out = true` for the acting user
   - Send a confirmation DM: "You've been unsubscribed from BadgBot messages."
2. In `lessonService.getDueLessons()`, add `AND u.opted_out = false` to the query
3. Register the action handler in `app.ts`

**Test:** Click Opt Out in Slack. Confirm `opted_out = true` in DB. Advance a lesson's `scheduled_for` to now — confirm no message is delivered to the opted-out user.

---

### Milestone 10 — Badge Retry on Failure

**Goal:** On startup, retry any badges that failed to send.

**Steps:**
1. In `app.ts`, after the app starts, query `badges WHERE awaiting_sent = true`
2. For each, re-send the congrats DM and set `awaiting_sent = false` on success

**Test:** Manually insert a badge row with `awaiting_sent = true`. Restart the app. Confirm the congrats DM is sent and the flag is cleared.

---

## Environment Variable Checklist

| Variable | Where to get it | Committed? |
|---|---|---|
| `SLACK_BOT_TOKEN` | Slack app → OAuth & Permissions | No — `.env` only |
| `SLACK_APP_TOKEN` | Slack app → Basic Information → App-Level Tokens (scope: `connections:write`) | No — `.env` only |
| `DATABASE_URL` | Your local PostgreSQL instance | No — `.env` only |

`.env` is blocked from git by `.gitignore`. To verify: `git check-ignore -v .env`

---

## Definition of Done

A milestone is complete when:
- [ ] TypeScript compiles with no errors (`npm run build`)
- [ ] The described manual test passes end-to-end in Slack
- [ ] No `.env` values are present in any committed file
