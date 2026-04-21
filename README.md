# Workout Backend API

> [Русская версия](README.ru.md)

A RESTful backend service for a fitness tracking application. Built with **Node.js / Express**, backed by **PostgreSQL** through the **Prisma ORM**, and secured with **JWT authentication**.

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Database Schema](#database-schema)
- [API Reference](#api-reference)
  - [Authentication](#authentication)
  - [Users](#users)
  - [Exercises](#exercises)
  - [Exercise Logs](#exercise-logs)
  - [Workouts](#workouts)
  - [Workout Logs](#workout-logs)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Contact](#contact)

---

## Overview

Workout Backend lets mobile or web clients:

- **Register and log in** - JWT-based auth with Argon2 password hashing.
- **Manage an exercise library** - create, update, and delete exercises with icon images.
- **Build workouts** - compose reusable workout plans from exercises.
- **Log training sessions** - start a workout log, track every set (weight × reps), mark sets and exercises complete, and finish the session.
- **Track progress** - each exercise log shows previous performance values so users can see improvement over time.
- **View statistics** - user profile endpoint aggregates total workouts and other summary data.

The application follows a layered architecture: routes → controllers → Prisma client → PostgreSQL. All protected routes require a valid JWT Bearer token.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js (ES Modules) |
| Framework | Express.js 4 |
| ORM | Prisma 4 |
| Database | PostgreSQL |
| Authentication | JWT (`jsonwebtoken`) |
| Password hashing | Argon2 |
| HTTP logging | Morgan |
| Code style | Prettier + `@trivago/prettier-plugin-sort-imports` |
| Dev server | Nodemon |

---

## Architecture

```
src/
├── app.js                  # Express app bootstrap & middleware stack
├── controllers/            # One folder per resource, one file per operation
│   ├── auth/               # register, login
│   ├── exercise/           # CRUD for exercise catalogue
│   ├── exersice-log/       # create, update, complete exercise log
│   ├── user/               # profile (with stats)
│   ├── workout/            # CRUD for workout plans
│   └── workout-log/        # create, update, complete workout session
├── db/
│   └── prisma.js           # Singleton Prisma client
├── middlewares/
│   ├── auth.js             # JWT protect middleware (attaches req.user)
│   └── error.js            # 404 + global error handler
├── routes/                 # Express Router files wired to controllers
└── utils/
    ├── calc-minutes.js     # Estimates workout duration (exercises × 5 min)
    ├── generate-token.js   # Signs a 10-day JWT
    ├── getUserWithoutPassword.js
    └── userDataWithoutPassword.js
prisma/
├── schema.prisma           # Data model & relations
└── migrations/             # SQL migration history (auto-generated)
public/
└── images/exercises/       # Static exercise icon files
```

**Middleware chain (app.js):**
Morgan → CORS → JSON body parser → static files → routes → 404 handler → error handler

---

## Database Schema

Five models, all backed by PostgreSQL:

### User
| Field | Type | Notes |
|---|---|---|
| id | Int (PK) | auto-increment |
| email | String | unique |
| name | String | |
| password | String | Argon2 hash |
| images | String[] | profile images |
| createdAt / updatedAt | DateTime | |

Relations: one-to-many with `ExerciseLog`, `WorkoutLog`.

### Exercise
| Field | Type | Notes |
|---|---|---|
| id | Int (PK) | auto-increment |
| name | String | |
| times | Int | default number of sets |
| icon | String | path to image file |
| createdAt / updatedAt | DateTime | |

Relations: many-to-many with `Workout`; one-to-many with `ExerciseLog`.

### Workout
| Field | Type | Notes |
|---|---|---|
| id | Int (PK) | auto-increment |
| name | String | |
| createdAt / updatedAt | DateTime | |

Relations: many-to-many with `Exercise`; one-to-many with `WorkoutLog`.

### WorkoutLog
| Field | Type | Notes |
|---|---|---|
| id | Int (PK) | auto-increment |
| isCompleted | Boolean | false by default |
| userId | Int | FK → User |
| workoutId | Int | FK → Workout |
| createdAt / updatedAt | DateTime | |

Relations: one-to-many with `ExerciseLog`.

### ExerciseLog
| Field | Type | Notes |
|---|---|---|
| id | Int (PK) | auto-increment |
| isCompleted | Boolean | false by default |
| userId | Int | FK → User |
| workoutLogId | Int | FK → WorkoutLog |
| exerciseId | Int | FK → Exercise |
| createdAt / updatedAt | DateTime | |

Relations: one-to-many with `ExerciseTime` (individual set records).

### ExerciseTime
| Field | Type | Notes |
|---|---|---|
| id | Int (PK) | auto-increment |
| weight | Float | default 0 |
| repeat | Int | default 0 |
| isCompleted | Boolean | false by default |
| exerciseLogId | Int | FK → ExerciseLog |
| createdAt / updatedAt | DateTime | |

---

## API Reference

All protected routes require the header:
```
Authorization: Bearer <token>
```

### Authentication

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/register` | - | Create account |
| POST | `/api/auth/login` | - | Log in |

**POST /api/auth/register**
```json
// Request body
{ "email": "user@example.com", "password": "secret", "name": "Alice" }

// Response 201
{ "user": { "id": 1, "email": "...", "name": "Alice", "images": [] }, "token": "<jwt>" }
```

**POST /api/auth/login**
```json
// Request body
{ "email": "user@example.com", "password": "secret" }

// Response 200
{ "user": { ... }, "token": "<jwt>" }
```

---

### Users

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/users/profile` | Required | Get profile + stats |

---

### Exercises

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/exercises` | Required | List all exercises (newest first) |
| POST | `/api/exercises` | Required | Create exercise |
| PUT | `/api/exercises/:id` | Required | Update exercise |
| DELETE | `/api/exercises/:id` | Required | Delete exercise |

**POST /api/exercises** body: `{ "name": "Push-up", "times": 3, "iconPath": "/images/pushup.png" }`

---

### Exercise Logs

An exercise log records one exercise within a workout session. Each log contains `ExerciseTime` rows (one per set).

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/exercises/log/:id` | Required | Get log with previous values |
| POST | `/api/exercises/log/:id` | Required | Create log for exercise `:id` |
| PUT | `/api/exercises/log/time/:id` | Required | Update a single set |
| PATCH | `/api/exercises/log/complete/:id` | Required | Mark log complete |

**PUT /api/exercises/log/time/:id** body: `{ "weight": 60, "repeat": 10, "isCompleted": true }`

**PATCH /api/exercises/log/complete/:id** body: `{ "isCompleted": true }`

---

### Workouts

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/workouts` | Required | List all workout plans |
| POST | `/api/workouts` | Required | Create workout plan |
| GET | `/api/workouts/:id` | Required | Get workout plan (incl. est. duration) |
| PUT | `/api/workouts/:id` | Required | Update workout plan |
| DELETE | `/api/workouts/:id` | Required | Delete workout plan |

**POST /api/workouts** body: `{ "name": "Push Day", "exercises": [1, 3, 7] }`

The `GET /api/workouts/:id` response includes `minutes` - estimated workout duration computed as `ceil(exerciseCount × 5)`.

---

### Workout Logs

A workout log is one completed training session. Creating it initialises exercise logs and sets for every exercise in the plan.

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/workouts/log/:id` | Required | Get session with all exercise logs |
| POST | `/api/workouts/log/:id` | Required | Start session for workout `:id` |
| PATCH | `/api/workouts/log/complete/:id` | Required | Mark session complete |

**PATCH /api/workouts/log/complete/:id** body: `{ "isCompleted": true }`

---

## Getting Started

### Prerequisites

- Node.js ≥ 18
- PostgreSQL database

### Installation

```bash
# 1. Clone the repo
git clone <repo-url>
cd workout-backend/back

# 2. Install dependencies
npm install

# 3. Create environment file
cp .env.example .env   # or create .env manually

# 4. Fill in .env
DATABASE_URL="postgresql://USER:PASSWORD@HOST:5432/DBNAME"
JWT_TOKEN="your-super-secret-key"
PORT=5000
NODE_ENV=development

# 5. Run database migrations
npx prisma migrate deploy

# 6. Start the dev server
npm run dev
```

The API will be available at `http://localhost:5000`.

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `JWT_TOKEN` | Yes | Secret for signing JWT tokens |
| `PORT` | No | HTTP port (default: 5000) |
| `NODE_ENV` | No | Set to `development` to enable Morgan logging |

---

## Project Structure

```
workout-backend/
├── back/                   # Application root
│   ├── package.json
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── migrations/
│   ├── public/
│   │   └── images/exercises/
│   └── src/
│       ├── app.js
│       ├── controllers/
│       ├── db/
│       ├── middlewares/
│       ├── routes/
│       └── utils/
└── README.md
```

---

## Contact

Oleg Melekh - [oleg.melekh.frontend@gmail.com](mailto:oleg.melekh.frontend@gmail.com)
