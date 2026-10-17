# TaskFlow

A multi-workspace project and task management web app. Teams can plan projects, assign and submit tasks, track progress on a live dashboard, and collaborate across separate workspaces.

**Stack:** PostgreSQL · Express · Node.js · React

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [System Architecture](#system-architecture)
- [Project Structure](#project-structure)
- [Database Schema](#database-schema)
- [API Overview](#api-overview)
- [Roles and Permissions](#roles-and-permissions)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Scripts](#scripts)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Features

### Authentication

- Sign up and sign in with email and password
- Email verification on signup
- JWT access token and refresh token (httpOnly cookie) with silent refresh
- Secure password hashing (bcrypt or argon2)

### Workspaces

- Create multiple workspaces
- Switch between workspaces from the sidebar or header
- Invite new members to a workspace via email
- Role-based access inside each workspace (owner, admin, member)

### Projects and Teams

- Create and manage projects inside a workspace
- Project list view with status and progress
- Create teams and add members to them
- Assign a team or individual members to a project

### Tasks

- Create, edit and delete tasks
- Assign tasks to workspace members
- Set priority (low, medium, high, urgent), status and due date
- Submit tasks for completion (with optional note or attachment link)
- Task list view with search, filters and sorting

### Dashboard

- Project overview
- Total tasks, completed tasks and overdue tasks
- Task summary
- Recent activity feed

### Calendar

- Month, week and day views
- Tasks appear on their due date, color-coded by priority
- Project start and due dates shown as markers
- Overdue tasks highlighted
- Click a task to open it, or click an empty date to create a task with that due date
- Filter by project, team or assignee ("My tasks" toggle)
- Follows the active workspace

### Analytics

- Tasks by priority
- Task completion rate over time

### Settings and UI

- Profile settings (name, avatar, password)
- Light and dark theme, remembered per user

---

## Tech Stack

| Layer       | Technology                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------- |
| Frontend    | React, React Router, Axios, Recharts (charts), FullCalendar or react-big-calendar (calendar), Tailwind CSS or CSS Modules |
| Backend     | Node.js, Express                                                                                                          |
| Database    | PostgreSQL                                                                                                                |
| Query layer | `pg` with plain SQL, or Prisma / Knex (choose one)                                                                        |
| Auth        | JWT (access and refresh tokens), bcrypt                                                                                   |
| Email       | Nodemailer (SMTP) or a provider such as SendGrid / SES                                                                    |
| Validation  | Zod or Joi                                                                                                                |

---

## System Architecture

```
┌──────────────┐     HTTPS / JSON     ┌───────────────────┐        ┌──────────────┐
│  React SPA   │ ───────────────────► │  Express API      │ ─────► │ PostgreSQL   │
│  (Vite)      │ ◄─────────────────── │  (Node.js)        │ ◄───── │              │
└──────────────┘                      └─────────┬─────────┘        └──────────────┘
                                                │
                                                ▼
                                       ┌──────────────────┐
                                       │ Email service    │
                                       │ (invites, verify)│
                                       └──────────────────┘
```

**Request flow**

1. The client sends a request with the access token in the `Authorization` header.
2. Auth middleware verifies the token and loads the user.
3. Workspace middleware checks that the user is a member of the workspace in the URL and attaches their role.
4. Role middleware allows or blocks the action.
5. The controller runs the query and returns JSON.

Every resource (projects, tasks, teams) belongs to a workspace, so every query is scoped by `workspace_id`.

---

## Project Structure

```
taskflow/
├── client/                     # React frontend
│   ├── src/
│   │   ├── api/                # Axios instance and API calls
│   │   ├── components/         # Reusable UI components
│   │   ├── context/            # Auth, Workspace, Theme providers
│   │   ├── hooks/
│   │   ├── pages/
│   │   │   ├── Auth/           # Signup, Signin, VerifyEmail
│   │   │   ├── Dashboard/
│   │   │   ├── Projects/
│   │   │   ├── Tasks/
│   │   │   ├── Teams/
│   │   │   ├── Calendar/
│   │   │   ├── Analytics/
│   │   │   └── Settings/
│   │   ├── routes/
│   │   ├── App.jsx
│   │   └── main.jsx
│   └── package.json
│
├── server/                     # Express backend
│   ├── src/
│   │   ├── config/             # DB connection, env loading
│   │   ├── controllers/
│   │   ├── middleware/         # auth, workspace, role, validate, error
│   │   ├── routes/
│   │   ├── services/           # Business logic, email service
│   │   ├── db/
│   │   │   ├── migrations/
│   │   │   └── seeds/
│   │   ├── utils/
│   │   ├── app.js
│   │   └── server.js
│   └── package.json
│
├── .gitignore
└── README.md
```

---

## Database Schema

Core tables (simplified):

```
users
  id (uuid, pk), name, email (unique), password_hash,
  avatar_url, email_verified, theme, created_at

refresh_tokens
  id, user_id (fk), token_hash, expires_at, revoked

workspaces
  id, name, owner_id (fk users), created_at

workspace_members
  id, workspace_id (fk), user_id (fk), role (owner | admin | member),
  joined_at, unique (workspace_id, user_id)

invitations
  id, workspace_id (fk), email, role, token_hash,
  invited_by (fk users), expires_at, accepted_at

projects
  id, workspace_id (fk), name, description, status,
  start_date, due_date, created_by (fk users), created_at

teams
  id, workspace_id (fk), name, description, created_at

team_members
  id, team_id (fk), user_id (fk), unique (team_id, user_id)

tasks
  id, workspace_id (fk), project_id (fk), title, description,
  priority (low | medium | high | urgent),
  status (todo | in_progress | submitted | completed),
  due_date, assignee_id (fk users), team_id (fk, nullable),
  created_by (fk users), submitted_at, completed_at, created_at

task_submissions
  id, task_id (fk), submitted_by (fk users), note, link, created_at

activity_logs
  id, workspace_id (fk), actor_id (fk users), action,
  entity_type, entity_id, metadata (jsonb), created_at
```

**Suggested indexes:** `tasks(workspace_id, status)`, `tasks(assignee_id)`, `tasks(due_date)`, `activity_logs(workspace_id, created_at DESC)`.

**Overdue tasks** are computed, not stored: `due_date < now() AND status != 'completed'`.

---

## API Overview

Base URL: `/api/v1`

### Auth

| Method | Endpoint             | Description                                        |
| ------ | -------------------- | -------------------------------------------------- |
| POST   | `/auth/signup`       | Create account and send verification email         |
| POST   | `/auth/verify-email` | Verify email with OTP or token                     |
| POST   | `/auth/signin`       | Sign in, returns access token, sets refresh cookie |
| POST   | `/auth/refresh`      | Get a new access token                             |
| POST   | `/auth/logout`       | Revoke refresh token                               |

### Workspaces

| Method | Endpoint                      | Description                |
| ------ | ----------------------------- | -------------------------- |
| GET    | `/workspaces`                 | List the user's workspaces |
| POST   | `/workspaces`                 | Create a workspace         |
| GET    | `/workspaces/:id`             | Workspace details          |
| POST   | `/workspaces/:id/invitations` | Invite a member by email   |
| POST   | `/invitations/accept`         | Accept an invitation       |
| GET    | `/workspaces/:id/members`     | List members               |

### Projects

| Method | Endpoint                   | Description      |
| ------ | -------------------------- | ---------------- |
| GET    | `/workspaces/:id/projects` | List projects    |
| POST   | `/workspaces/:id/projects` | Create a project |
| PATCH  | `/projects/:projectId`     | Update a project |
| DELETE | `/projects/:projectId`     | Delete a project |

### Tasks

| Method | Endpoint                | Description                                                |
| ------ | ----------------------- | ---------------------------------------------------------- |
| GET    | `/workspaces/:id/tasks` | List tasks (filter by status, priority, assignee, project) |
| POST   | `/workspaces/:id/tasks` | Create a task                                              |
| PATCH  | `/tasks/:taskId`        | Update a task                                              |
| PATCH  | `/tasks/:taskId/assign` | Assign a task                                              |
| POST   | `/tasks/:taskId/submit` | Submit a task                                              |
| DELETE | `/tasks/:taskId`        | Delete a task                                              |

### Teams

| Method | Endpoint                 | Description            |
| ------ | ------------------------ | ---------------------- |
| GET    | `/workspaces/:id/teams`  | List teams             |
| POST   | `/workspaces/:id/teams`  | Create a team          |
| POST   | `/teams/:teamId/members` | Add a member to a team |

### Calendar

| Method | Endpoint                                                 | Description                                                                                        |
| ------ | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| GET    | `/workspaces/:id/calendar?from=YYYY-MM-DD&to=YYYY-MM-DD` | Tasks and project deadlines in a date range (optional `projectId`, `teamId`, `assigneeId` filters) |

The calendar needs no extra table. It queries `tasks.due_date` and `projects.start_date / due_date` for the visible range.

### Dashboard and Analytics

| Method | Endpoint                               | Description                                  |
| ------ | -------------------------------------- | -------------------------------------------- |
| GET    | `/workspaces/:id/dashboard`            | Totals, completed, overdue, project overview |
| GET    | `/workspaces/:id/activity`             | Recent activity                              |
| GET    | `/workspaces/:id/analytics/priority`   | Task counts by priority                      |
| GET    | `/workspaces/:id/analytics/completion` | Completion trend over time                   |

### User

| Method | Endpoint             | Description              |
| ------ | -------------------- | ------------------------ |
| GET    | `/users/me`          | Current user profile     |
| PATCH  | `/users/me`          | Update profile and theme |
| PATCH  | `/users/me/password` | Change password          |

---

## Roles and Permissions

| Action                                               | Owner | Admin |         Member          |
| ---------------------------------------------------- | :---: | :---: | :---------------------: |
| Delete workspace                                     |  Yes  |  No   |           No            |
| Invite members                                       |  Yes  |  Yes  |           No            |
| Create projects and teams                            |  Yes  |  Yes  |           No            |
| Create and assign tasks                              |  Yes  |  Yes  | Yes (own or unassigned) |
| Delete tasks                                         |  Yes  |  Yes  |     Own tasks only      |
| Submit assigned tasks                                |  Yes  |  Yes  |           Yes           |
| View dashboard, calendar and analytics               |  Yes  |  Yes  |           Yes           |
| Change a task's due date by dragging on the calendar |  Yes  |  Yes  |     Own tasks only      |

---

## Getting Started

### Prerequisites

- Node.js 18 or newer
- PostgreSQL 14 or newer
- npm or yarn

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/taskflow.git
cd taskflow
```

### 2. Set up the database

```bash
psql -U postgres
CREATE DATABASE taskflow;
\q
```

### 3. Set up the backend

```bash
cd server
npm install
cp .env.example .env     # then fill in the values
npm run migrate          # create tables
npm run seed             # optional demo data
npm run dev
```

The API runs on `http://localhost:5000`.

### 4. Set up the frontend

```bash
cd client
npm install
cp .env.example .env
npm run dev
```

The app runs on `http://localhost:5173`.

---

## Environment Variables

### `server/.env`

```env
NODE_ENV=development
PORT=5000

DATABASE_URL=postgresql://postgres:password@localhost:5432/taskflow

JWT_ACCESS_SECRET=change_me
JWT_REFRESH_SECRET=change_me_too
ACCESS_TOKEN_EXPIRES_IN=15m
REFRESH_TOKEN_EXPIRES_IN=30d

CLIENT_URL=http://localhost:5173

SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=your_user
SMTP_PASS=your_password
EMAIL_FROM="TaskFlow <no-reply@taskflow.com>"
```

### `client/.env`

```env
VITE_API_URL=http://localhost:5000/api/v1
```

> Never commit `.env` files. Make sure `.env` is listed in `.gitignore`.

---

## Scripts

### Server

| Command           | Description              |
| ----------------- | ------------------------ |
| `npm run dev`     | Start with nodemon       |
| `npm start`       | Start in production mode |
| `npm run migrate` | Run database migrations  |
| `npm run seed`    | Insert demo data         |
| `npm test`        | Run tests                |

### Client

| Command           | Description                  |
| ----------------- | ---------------------------- |
| `npm run dev`     | Start dev server             |
| `npm run build`   | Production build             |
| `npm run preview` | Preview the production build |
| `npm run lint`    | Lint the code                |

---

## Roadmap

**Phase 1: Foundation**

- [ ] Project setup, database migrations
- [ ] Signup, signin, email verification, token refresh
- [ ] Workspace creation and switching

**Phase 2: Core**

- [ ] Projects and teams
- [ ] Task CRUD, assignment and submission
- [ ] Email invitations

**Phase 3: Insights**

- [ ] Dashboard (totals, completed, overdue, overview)
- [ ] Recent activity and task summary
- [ ] Calendar (month/week/day views, task and project deadlines)
- [ ] Analytics (priority and completion charts)

**Phase 4: Polish**

- [ ] Profile settings
- [ ] Light and dark theme
- [ ] Responsive layout, loading and error states
- [ ] Tests and deployment

**Future ideas:** custom calendar events and meetings, Google Calendar / iCal sync, due-date reminder emails, comments on tasks, file attachments, real-time updates with WebSockets, in-app notifications, Kanban board view.

---
