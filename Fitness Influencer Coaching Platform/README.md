# Fitness Coaching Platform — Database Design

An online coaching ecosystem where trainers/influencers manage multiple clients and provide structured fitness support including plans, sessions, check-ins, progress tracking, and payments.

---

[fitness-training-centre](./fitness.png)

## Table of Contents

- [Overview](#overview)
- [Entities](#entities)
- [Entity Descriptions](#entity-descriptions)
- [Relationships & Cardinality](#relationships--cardinality)
- [Key Design Decisions](#key-design-decisions)
- [ERD (Eraser.io)](#erd-eraserio)

---

## Overview

This database supports:

- Multiple trainers managing multiple clients
- Selling and subscribing to fitness plans (workout, diet, combo)
- Scheduling one-on-one sessions and consultations
- Weekly client check-ins with trainer feedback
- Progress tracking (weight, measurements, body fat)
- Payment and subscription management

---

## Entities

| Entity             | Purpose                                                                            |
| ------------------ | ---------------------------------------------------------------------------------- |
| `users`            | All platform users — trainers and clients share one table, distinguished by `role` |
| `trainer_profiles` | Trainer-specific data: bio, specialization, experience                             |
| `client_profiles`  | Client-specific data: fitness goal, health notes, height                           |
| `trainer_client`   | Maps which trainer is assigned to which client (many-to-many)                      |
| `plans`            | Coaching programs created by trainers (workout / diet / combo / consultation)      |
| `plan_content`     | Week-by-week content inside a plan (exercises, meals, notes)                       |
| `enrollments`      | Records a client purchasing/subscribing to a plan — the core join table            |
| `payments`         | Payment record linked to each enrollment                                           |
| `sessions`         | Scheduled video calls or live sessions between trainer and client                  |
| `check_ins`        | Weekly self-reports submitted by clients, with trainer feedback                    |
| `progress`         | Numerical body measurements captured during a check-in                             |

---

## Entity Descriptions

### `users`

The base identity table for everyone on the platform. Both trainers and clients are stored here. The `role` field (`"trainer"` or `"client"`) determines what profile and permissions apply.

```
id              uuid        PK
full_name       string
email           string      UNIQUE
phone           string
role            string      -- "trainer" | "client"
profile_photo_url string
created_at      timestamp
```

---

### `trainer_profiles`

Extended profile for users who are trainers.

```
id                uuid    PK
user_id           uuid    FK → users.id
bio               text
specialization    string
instagram_handle  string
years_experience  int
```

---

### `client_profiles`

Extended profile for users who are clients.

```
id              uuid    PK
user_id         uuid    FK → users.id
date_of_birth   date
gender          string
height_cm       float
fitness_goal    string
health_notes    text
```

---

### `trainer_client`

Resolves the many-to-many relationship between trainers and clients. A trainer can coach many clients; a client could theoretically have co-coaches.

```
id            uuid        PK
trainer_id    uuid        FK → users.id
client_id     uuid        FK → users.id
status        string      -- "active" | "paused" | "ended"
assigned_at   timestamp
```

---

### `plans`

A catalog of coaching programs created by a trainer. Plans are templates — clients subscribe to them via `enrollments`.

```
id              uuid      PK
created_by      uuid      FK → users.id
title           string
type            string    -- "workout" | "diet" | "combo" | "consultation"
description     text
duration_weeks  int
price           decimal
is_active       boolean
created_at      timestamp
```

---

### `plan_content`

The actual week-by-week material inside a plan. Separated from `plans` to keep the catalog lean and content queryable independently.

```
id              uuid    PK
plan_id         uuid    FK → plans.id
content_type    string  -- "workout" | "diet" | "note"
title           string
body            text
week_number     int
day_number      int
```

---

### `enrollments`

The most important table. Created when a client purchases a plan. Tracks start/end dates and status. A client can enroll in the same plan multiple times (e.g. after completing and re-subscribing).

```
id            uuid        PK
client_id     uuid        FK → users.id
plan_id       uuid        FK → plans.id
trainer_id    uuid        FK → users.id
start_date    date
end_date      date
status        string      -- "active" | "completed" | "cancelled"
enrolled_at   timestamp
```

---

### `payments`

One payment record per enrollment. Linked to the enrollment rather than directly to a user or plan, so it captures the exact purchase event.

```
id                uuid        PK
enrollment_id     uuid        FK → enrollments.id   (1-to-1)
client_id         uuid        FK → users.id
amount            decimal
currency          string
status            string      -- "paid" | "pending" | "failed" | "refunded"
payment_method    string
transaction_ref   string
paid_at           timestamp
```

---

### `sessions`

Scheduled video calls or live coaching sessions. Separate from check-ins — a session is trainer-initiated, has a meeting link, and is time-boxed.

```
id                uuid        PK
trainer_id        uuid        FK → users.id
client_id         uuid        FK → users.id
enrollment_id     uuid        FK → enrollments.id
session_type      string      -- "consultation" | "follow_up" | "live"
scheduled_at      timestamp
duration_minutes  int
meet_link         string
status            string      -- "scheduled" | "completed" | "cancelled"
trainer_notes     text
```

---

### `check_ins`

Weekly self-reports submitted by the client. Not the same as a session — no meeting link, no scheduling. The trainer responds via `trainer_feedback`. Linked to an enrollment to track which plan period the check-in belongs to.

```
id                  uuid    PK
client_id           uuid    FK → users.id
enrollment_id       uuid    FK → enrollments.id
week_number         int
submitted_on        date
client_notes        text
photo_url           string
trainer_feedback    text
feedback_given_at   timestamp
```

---

### `progress`

Numerical body measurements captured with each check-in. Kept in its own table so measurements can be queried and charted independently without touching check-in text data.

```
id              uuid    PK
client_id       uuid    FK → users.id
check_in_id     uuid    FK → check_ins.id   (1-to-1)
recorded_on     date
weight_kg       float
chest_cm        float
waist_cm        float
hips_cm         float
body_fat_pct    float
```

---

## Relationships & Cardinality

| Relationship                           | Type     | Notes                                        |
| -------------------------------------- | -------- | -------------------------------------------- |
| `users` → `trainer_profiles`           | 1 : 0..1 | A user may or may not be a trainer           |
| `users` → `client_profiles`            | 1 : 0..1 | A user may or may not be a client            |
| `users` ↔ `users` via `trainer_client` | M : N    | One trainer coaches many clients             |
| `users` → `plans`                      | 1 : N    | A trainer creates many plans                 |
| `plans` → `plan_content`               | 1 : N    | A plan has many content items                |
| `plans` ↔ `users` via `enrollments`    | M : N    | Many clients can enroll in one plan          |
| `enrollments` → `payments`             | 1 : 1    | Each enrollment has one payment record       |
| `enrollments` → `sessions`             | 1 : N    | An enrollment can have many sessions         |
| `enrollments` → `check_ins`            | 1 : N    | An enrollment has many weekly check-ins      |
| `check_ins` → `progress`               | 1 : 1    | Each check-in captures one progress snapshot |

---

## Key Design Decisions

**Single `users` table with `role` column**
Trainers and clients share authentication and contact info. Role-specific data is offloaded to `trainer_profiles` and `client_profiles` to avoid NULL-heavy columns.

**`enrollments` as the central join table**
Rather than a simple M:N link, `enrollments` carries its own state (`start_date`, `end_date`, `status`). This lets a client enroll in the same plan more than once over time, and scopes sessions and check-ins to a specific plan period.

**`sessions` ≠ `check_ins`**
Sessions are trainer-scheduled video calls with a meeting link and duration. Check-ins are client-submitted weekly text/photo reports. They serve different purposes and must remain separate tables.

**`progress` separate from `check_ins`**
Numerical measurements (weight, waist, body fat %) are analytical data. Separating them allows charting and querying progress over time without loading check-in text fields.

**`plan_content` separate from `plans`**
`plans` is the catalog (title, price, duration). `plan_content` is the actual material. This separation keeps plan listings lightweight and lets content be queried by week or day independently.

**`payments` linked to `enrollments`, not `users`**
A payment is always for a specific enrollment event, not just a general user transaction. This preserves full context: which plan, which client, which period.

---

## ERD (Eraser.io)

Paste the code below into [eraser.io](https://eraser.io) using the **Entity Relationship Diagram** template.

```
users [icon: user, color: blue] {
  id uuid pk
  full_name string
  email string unique
  phone string
  role string
  profile_photo_url string
  created_at timestamp
}

trainer_profiles [icon: star, color: blue] {
  id uuid pk
  user_id uuid fk
  bio text
  specialization string
  instagram_handle string
  years_experience int
}

client_profiles [icon: user, color: green] {
  id uuid pk
  user_id uuid fk
  date_of_birth date
  gender string
  height_cm float
  fitness_goal string
  health_notes text
}

trainer_client [icon: users, color: yellow] {
  id uuid pk
  trainer_id uuid fk
  client_id uuid fk
  status string
  assigned_at timestamp
}

plans [icon: file-text, color: orange] {
  id uuid pk
  created_by uuid fk
  title string
  type string
  description text
  duration_weeks int
  price decimal
  is_active boolean
  created_at timestamp
}

plan_content [icon: list, color: orange] {
  id uuid pk
  plan_id uuid fk
  content_type string
  title string
  body text
  week_number int
  day_number int
}

enrollments [icon: clipboard, color: purple] {
  id uuid pk
  client_id uuid fk
  plan_id uuid fk
  trainer_id uuid fk
  start_date date
  end_date date
  status string
  enrolled_at timestamp
}

payments [icon: credit-card, color: red] {
  id uuid pk
  enrollment_id uuid fk
  client_id uuid fk
  amount decimal
  currency string
  status string
  payment_method string
  transaction_ref string
  paid_at timestamp
}

sessions [icon: video, color: teal] {
  id uuid pk
  trainer_id uuid fk
  client_id uuid fk
  enrollment_id uuid fk
  session_type string
  scheduled_at timestamp
  duration_minutes int
  meet_link string
  status string
  trainer_notes text
}

check_ins [icon: check-square, color: green] {
  id uuid pk
  client_id uuid fk
  enrollment_id uuid fk
  week_number int
  submitted_on date
  client_notes text
  photo_url string
  trainer_feedback text
  feedback_given_at timestamp
}

progress [icon: trending-up, color: pink] {
  id uuid pk
  client_id uuid fk
  check_in_id uuid fk
  recorded_on date
  weight_kg float
  chest_cm float
  waist_cm float
  hips_cm float
  body_fat_pct float
}

// Relationships
users.id < trainer_profiles.user_id
users.id < client_profiles.user_id
users.id < trainer_client.trainer_id
users.id < trainer_client.client_id
users.id < plans.created_by
users.id < enrollments.client_id
users.id < enrollments.trainer_id
users.id < sessions.trainer_id
users.id < sessions.client_id
users.id < check_ins.client_id
users.id < progress.client_id
plans.id < plan_content.plan_id
plans.id < enrollments.plan_id
enrollments.id - payments.enrollment_id
enrollments.id < sessions.enrollment_id
enrollments.id < check_ins.enrollment_id
check_ins.id - progress.check_in_id
```
