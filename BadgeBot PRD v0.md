# PRD: BadgBot — Slack Badge Delivery System

## Overview

BadgBot is a Slack bot that delivers daily learning content to enrolled users and automatically awards a badge when a user completes all lessons in a module. It operates entirely within Slack using Socket Mode and is backed by a relational database.

---

## Problem Statement

Learning programs delivered asynchronously suffer from low completion rates and no visible recognition. Learners need daily nudges, a low-friction way to reflect on what they've learned, and a meaningful reward when they finish. BadgBot solves all three without requiring learners to leave Slack.

---

## Users

| Role | Description |
|---|---|
| **Learner** | Enrolled in one or more modules; receives daily lessons and earns badges |
| **Admin/Facilitator** | Creates modules and lessons; monitors completion (out of scope for v1) |

---

## Core Concepts

- **Module** — a named learning track with a description, an associated Slack channel, and a fixed number of lessons
- **Lesson** — one day's content within a module; has content blocks, optional materials, and a scheduled delivery time
- **Enrollment** — the join between a user and a module; tracks certificate issuance
- **Badge** — awarded to a user upon module completion; has an issued timestamp and a sent flag
- **Reflection** — a user's written response to a lesson; can be public (posted to a Slack thread) or private

---

## User Flow

### 1. Daily Lesson Delivery
BadgBot sends a formatted Slack message to each enrolled user (or the module's channel) at the lesson's `scheduled_for` time. The message contains the lesson's content blocks and any linked materials.

### 2. Reflection Submission
The lesson message includes a **"Submit Reflection"** button. Clicking it opens a Slack modal with:
- A text area for the reflection
- A public/private toggle
- Submit and Cancel buttons

On submit, BadgBot saves the reflection to `theory_submissions`. If marked public, it posts the reflection text to the module's designated Slack thread (`reflection_threads`).

### 3. Module Completion & Badge Award
When a user submits their final lesson reflection, BadgBot checks whether all lessons in the module are complete. If yes:
- A new row is inserted into `badges` with `awaiting_sent = true`
- BadgBot sends a congratulations message in Slack with the badge and module summary
- The `badges.awaiting_sent` flag is set to `false` after successful delivery
- `enrollments.certificate_id` and `issued_at` are populated

---

## Functional Requirements

### Lesson Delivery
- [ ] Query lessons where `scheduled_for` is due and the user is enrolled
- [ ] Send formatted message to user (or channel) with content blocks and materials
- [ ] Store `daily_message_id` on the lesson record for threading

### Reflection Modal
- [ ] Trigger modal on button click in lesson message
- [ ] Validate reflection text is not empty before allowing submit
- [ ] Save to `theory_submissions` with `user_id`, `lesson_id`, `reflection_text`, `is_public`, `submitted_at`
- [ ] If `is_public`, post to the module's reflection thread; save `thread_message_id`

### Badge Issuance
- [ ] On each reflection submit, check if all lessons for the module now have a submission from this user
- [ ] If complete: insert badge record, send congratulations message, update enrollment record
- [ ] Handle `awaiting_sent` flag to support retry if Slack delivery fails

### Opt-Out
- [ ] Users can opt out via a command or button; sets `users.opted_out = true`
- [ ] Opted-out users receive no further messages

---

## Data Model Summary

| Table | Purpose |
|---|---|
| `users` | Slack identity and opt-out status |
| `modules` | Learning tracks with Slack channel binding |
| `lessons` | Daily content units per module |
| `module_enrollments` | User ↔ module enrollment records |
| `enrollments` | Certificate/badge issuance state per user per module |
| `badges` | Issued badges with delivery tracking flag |
| `theory_submissions` | Per-lesson reflections with public/private flag |
| `reflection_threads` | Slack thread anchors for public reflections per module |

---

## Out of Scope (v1)

- Admin UI for creating modules/lessons (assumed seeded directly to DB)
- Leaderboards or social badge display
- Multiple badges per module
- Editing or deleting a submitted reflection

---

## Success Metrics

- % of enrolled users who complete all lessons in a module
- % of reflections marked public (engagement signal)
- Badge delivery success rate (`awaiting_sent` flips to false within 60s)
