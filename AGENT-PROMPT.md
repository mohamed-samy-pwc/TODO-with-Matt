# Agent Prompt: Build "Flux" — A Next-Generation TODO Web App

---

## Project Identity

**App name:** Flux  
**Tagline:** *Your tasks. Your flow. Uninterrupted.*  
**Concept:** A drag-and-drop-first, AI-aware, multi-view task manager that feels like the love child of Linear's speed, TickTick's richness, and Notion's flexibility — but built by a designer who actually uses their own product.

---

## Tech Stack (Non-Negotiable)

### Frontend
- **Framework:** Next.js 15 (App Router, React Server Components where appropriate)
- **Language:** TypeScript (strict mode — no `any`)
- **Styling:** Tailwind CSS v4 + shadcn/ui for base components
- **Drag and Drop:** `dnd-kit` (`@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`) — **do not use react-beautiful-dnd (archived)**
- **Animations:** Framer Motion for card transitions, view switches, and drag feedback
- **State:** Zustand for client state + TanStack Query v5 for server state / caching
- **Forms:** React Hook Form + Zod validation
- **Icons:** Lucide React (consistent set, no mixing icon libraries)
- **Date handling:** date-fns v3
- **Theming:** next-themes for light/dark/system

### Backend
- **Framework:** Spring Boot 3.x
- **Language:** Kotlin (data classes for DTOs, extension functions, coroutines where async)
- **Auth:** Spring Security with JWT (RS256, auto key rotation via JWKS)
- **Database:** PostgreSQL with Spring Data JPA + Hibernate
- **Real-time:** Spring WebSocket with STOMP protocol for live sync across tabs/devices
- **Migrations:** Flyway
- **Testing:** JUnit 5 + MockK + Testcontainers for integration tests
- **Build:** Gradle with Kotlin DSL

### Deployment (Free Tier — Professional)
- **Frontend:** Vercel (Hobby plan — free, custom domain support, automatic preview deployments per PR)
- **Backend:** Fly.io free tier (shared-cpu-1x / 512MB RAM) via Dockerfile — always-on, no cold-start spin-down like Render
- **Database:** Supabase free tier (PostgreSQL) — 500MB, connection pooling via PgBouncer included
- **CI/CD:** GitHub Actions — lint + test + build on every PR; auto-deploy to Vercel (frontend) and Fly.io (backend) on merge to `main`

---

## Repository Structure

```
flux/
├── apps/
│   ├── web/                          # Next.js frontend
│   │   ├── app/
│   │   │   ├── (auth)/               # Login / register routes
│   │   │   ├── (dashboard)/          # Authenticated app shell
│   │   │   │   ├── board/            # Kanban view
│   │   │   │   ├── list/             # List view
│   │   │   │   ├── calendar/         # Calendar view
│   │   │   │   ├── matrix/           # Eisenhower Matrix view
│   │   │   │   └── focus/            # Focus / Pomodoro mode
│   │   │   └── api/                  # Next.js API routes (proxies / webhooks only)
│   │   ├── components/
│   │   │   ├── board/                # Kanban-specific components
│   │   │   ├── cards/                # Task card variants
│   │   │   ├── dnd/                  # dnd-kit drag overlay, sensors, strategies
│   │   │   ├── ui/                   # shadcn/ui extended components
│   │   │   └── views/               # View-switcher shell
│   │   └── stores/                   # Zustand stores
│   └── backend/                      # Spring Boot Kotlin app
│       ├── src/main/kotlin/dev/flux/
│       │   ├── auth/                 # JWT config, filters, user details service
│       │   ├── tasks/                # Task entity, repo, service, controller
│       │   ├── boards/               # Board/Column entity, repo, service, controller
│       │   ├── users/                # User entity + profile
│       │   ├── realtime/             # WebSocket STOMP config + message handlers
│       │   ├── ai/                   # NLP parsing service (OpenAI-compatible endpoint)
│       │   └── config/               # Security, CORS, Flyway, Jackson config
│       ├── src/main/resources/
│       │   ├── db/migration/         # Flyway SQL migrations (V1__, V2__, etc.)
│       │   └── application.yml
│       └── Dockerfile
├── .github/
│   └── workflows/
│       ├── ci.yml                    # Lint + test on PRs
│       └── deploy.yml                # Deploy on merge to main
├── fly.toml                          # Fly.io backend config
└── README.md
```

---

## Database Schema

Design the following tables (via Flyway migrations):

```sql
-- V1__init.sql
users (id UUID PK, email TEXT UNIQUE, password_hash TEXT, display_name TEXT, avatar_url TEXT, created_at TIMESTAMPTZ)
boards (id UUID PK, user_id UUID FK, title TEXT, color TEXT, icon TEXT, position INT, created_at TIMESTAMPTZ)
columns (id UUID PK, board_id UUID FK, title TEXT, color TEXT, position INT, wip_limit INT NULLABLE)
tasks (
  id UUID PK,
  column_id UUID FK,
  board_id UUID FK,
  user_id UUID FK,
  title TEXT NOT NULL,
  description TEXT,
  priority SMALLINT DEFAULT 0,       -- 0=none, 1=low, 2=medium, 3=high, 4=urgent
  status TEXT DEFAULT 'todo',        -- todo | in_progress | done | archived
  due_date TIMESTAMPTZ NULLABLE,
  reminder_at TIMESTAMPTZ NULLABLE,
  position FLOAT NOT NULL,           -- lexicographic ordering (LexoRank-style)
  tags TEXT[] DEFAULT '{}',
  is_pinned BOOLEAN DEFAULT FALSE,
  recurrence_rule TEXT NULLABLE,     -- RFC 5545 RRULE string
  estimated_minutes INT NULLABLE,
  actual_minutes INT NULLABLE,
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ
)
subtasks (id UUID PK, task_id UUID FK, title TEXT, is_done BOOLEAN, position INT)
comments (id UUID PK, task_id UUID FK, user_id UUID FK, body TEXT, created_at TIMESTAMPTZ)
habits (id UUID PK, user_id UUID FK, title TEXT, color TEXT, icon TEXT, target_days TEXT[], streak INT DEFAULT 0)
habit_completions (id UUID PK, habit_id UUID FK, completed_on DATE)
pomodoro_sessions (id UUID PK, task_id UUID FK NULLABLE, user_id UUID FK, started_at TIMESTAMPTZ, ended_at TIMESTAMPTZ, type TEXT)
```

---

## Backend API Design

### Authentication
```
POST /api/auth/register     { email, password, displayName } → { token, user }
POST /api/auth/login        { email, password } → { token, user }
POST /api/auth/refresh      { refreshToken } → { token }
GET  /api/auth/me           → User profile
```

### Boards & Columns
```
GET    /api/boards                   → List user's boards
POST   /api/boards                   → Create board
PATCH  /api/boards/{id}              → Update title/color/icon
DELETE /api/boards/{id}              → Delete board
POST   /api/boards/{id}/columns      → Create column
PATCH  /api/columns/{id}             → Update column
DELETE /api/columns/{id}             → Delete column
PATCH  /api/boards/{id}/columns/reorder  → Bulk reorder columns (position array)
```

### Tasks
```
GET    /api/tasks?boardId=&columnId=&status=&tag=&due=  → Filtered task list
POST   /api/tasks                    → Create task
PATCH  /api/tasks/{id}               → Update any field
DELETE /api/tasks/{id}               → Soft delete (archived)
PATCH  /api/tasks/reorder            → Bulk move/reorder: [{ id, columnId, position }]
GET    /api/tasks/{id}/subtasks      → List subtasks
POST   /api/tasks/{id}/subtasks      → Add subtask
PATCH  /api/subtasks/{id}            → Toggle done / rename
DELETE /api/subtasks/{id}            → Remove
GET    /api/tasks/{id}/comments      → Thread
POST   /api/tasks/{id}/comments      → Add comment
```

### Smart Features
```
POST /api/tasks/parse-natural        { text: "call dentist next Thursday 3pm high priority" } → ParsedTask
GET  /api/tasks/overdue              → Overdue tasks with suggested reschedule
GET  /api/tasks/today                → Smart "Today" view (due today + pinned + in-progress)
GET  /api/tasks/upcoming             → Next 7 days aggregated
GET  /api/analytics/heatmap          → Daily completion counts (last 365 days)
GET  /api/analytics/by-project       → Completion rate per board
```

### Habits
```
GET    /api/habits                   → All habits
POST   /api/habits                   → Create
PATCH  /api/habits/{id}              → Update
POST   /api/habits/{id}/complete     → Mark today done
GET    /api/habits/{id}/stats        → Streak, completion rate, heatmap data
```

### Real-Time (WebSocket)
```
CONNECT  ws://api/ws                 (STOMP, Bearer token in headers)
SUBSCRIBE /user/queue/tasks          → Live task updates pushed to user
SUBSCRIBE /topic/board/{boardId}     → Board-level updates (for team boards future)
SEND /app/task.update                → Optimistic update confirmation
```

---

## Frontend Features — Detailed Specification

### 1. Multi-View Dashboard

The app has **four primary views** on the same task data, toggled via a top-bar segmented control:

#### (A) Kanban Board View (Default)
- Columns rendered horizontally with smooth scroll
- Tasks rendered as draggable cards via `dnd-kit`
- Drag a card: it lifts with `scale(1.04)` + drop shadow, other cards animate out of the way with spring physics (Framer Motion `layoutId`)
- Drag between columns: column highlights with a colored drop zone border
- Drag to reorder columns themselves (drag column header)
- WIP Limit badge: if a column has a WIP limit set and cards exceed it, the column header turns amber with a warning icon
- `+ Add Task` button at bottom of each column — inline text input, no modal for quick add
- Column context menu: rename, set WIP limit, set color, delete, add column left/right
- "Add Column" button at the end of the board

#### (B) List View
- Grouped by column/status, collapsible sections
- Sortable by: due date, priority, created date, title (click column header)
- Drag rows to reorder — same dnd-kit sortable strategy
- Inline edit: click any field in the row (title, due date, priority pill) to edit in-place
- Bulk select: checkbox column — select multiple, then bulk-move/delete/tag
- Filter bar at top: filter by priority, tags, due date range, assignee (future)

#### (C) Calendar View
- Monthly/weekly toggle
- Tasks with due dates appear on their date as colored chips
- Drag a chip to a different date → updates due date via API
- Day cell click → quick-add inline input pre-populated with that date
- Overdue tasks shown in red with a strikethrough-style indicator

#### (D) Eisenhower Matrix View
- 2×2 grid: Urgent+Important / Not Urgent+Important / Urgent+Not Important / Not Urgent+Not Important
- Task priority maps to quadrant automatically
- Drag card between quadrants → updates priority
- Each quadrant has its own scroll if overflowing

---

### 2. Task Cards — Design & Interaction

Each Kanban card is a **rich card** (not a plain text row). Design:

```
┌──────────────────────────────────────┐
│ 🔴 URGENT  [#design] [#ux]           │  ← Priority badge + tag chips
│                                      │
│  Redesign onboarding flow            │  ← Title (truncated 2 lines)
│  Complete wireframes for 3 screens   │  ← Description preview (1 line, muted)
│                                      │
│  ████████░░ 4/5 subtasks done       │  ← Subtask progress bar
│                                      │
│  📅 Nov 3    🕐 2h est   💬 3       │  ← Due date | estimate | comment count
└──────────────────────────────────────┘
```

- **Card colors:** Left border accent color matches the column color (4px)
- **Priority badge:** color-coded pill: red=urgent, orange=high, blue=medium, gray=low
- **Tag chips:** colored mini-pills, max 2 visible + "+N more"
- **Subtask progress bar:** thin bar at bottom interior of card, fills green as subtasks complete
- **Overdue indicator:** due date turns red, card gets a subtle red glow on hover
- **Pinned cards:** thumbtack icon top-right, card has a slightly different background
- **Card hover state:** subtle elevation lift + quick-action toolbar appears: ✏️ Edit | 📋 Copy | 🗑 Delete | ⭐ Pin
- **Card click → Slide-in Detail Panel** (right drawer, 480px wide, not a modal) showing full task detail

#### Task Detail Drawer
- Full title (editable inline)
- Rich text description (Tiptap editor — bold, italic, bullet lists, code blocks)
- Priority selector (segmented control)
- Due date picker (date + time)
- Reminder picker
- Tags (multi-select + create new)
- Estimated time input
- Subtask list (checklist, drag to reorder, inline add)
- Recurrence rule builder (daily / weekly / monthly / custom RRULE)
- Comments thread at bottom
- Activity log (created, updated, moved) in muted text
- "Focus on this task" button → enters Focus Mode

---

### 3. Natural Language Quick Add

A global command bar (triggered by `Cmd+K` or `Ctrl+K`):

- Floating centered modal with a text input
- As you type, it calls `POST /api/tasks/parse-natural` (debounced 300ms)
- Right side shows live parsed preview:
  - Detected title, due date, priority, tags
  - "Creating in: [Board] > [Column]" with a dropdown to change target
- Press Enter → creates task, shows success toast with "Undo" action (3s)
- Also surfaces recent tasks, boards, and commands (switch view, open settings)

Example: `"finish slide deck Friday 9am urgent #work"` parses to:
- Title: "finish slide deck"
- Due: this Friday 09:00
- Priority: urgent
- Tags: [work]

Backend NLP parsing: Use a regex + keyword parser first (free, no API cost). Detect:
- Date/time patterns: tomorrow, next Monday, [day] at [time], in 2 hours, etc. (use `natty` or custom Kotlin parser)
- Priority keywords: urgent, asap, important, high, low, critical
- Tags: `#word` anywhere in string
- Time estimates: "2h", "30min", "~1 hour"

---

### 4. Focus Mode (Pomodoro)

Route: `/focus`

Full-screen minimal UI:
- Current task title centered
- Circular progress ring (SVG animated) counting down 25:00
- Below: session type indicator (Work / Short Break / Long Break) + session count (e.g., "Session 3 of 4")
- Ambient background: slow-shifting gradient (purple → blue → teal, CSS animation)
- Minimal controls: Play/Pause, Skip, Stop
- Bottom: mini task list — click to switch focus task
- On session complete: browser notification + gentle chime sound (Web Audio API)
- Sessions logged to backend `pomodoro_sessions` table
- Stats shown after stopping: total focus time, tasks completed this session

---

### 5. Habit Tracker

Integrated into the sidebar, not a separate app:
- List of habits with icons and colored left borders
- Today's habits: each shows a checkbox — tap to mark done
- **Streak counter** next to each habit (flame emoji + count)
- Click habit → detail modal: GitHub-style contribution heatmap (365-day grid), weekly completion rate chart (recharts)
- "Add Habit" → name, icon picker (Lucide icons), color, repeat days (Mon-Sun checkboxes or "Every day")
- Habits completing today give a micro-celebration animation (confetti burst, 0.5s)

---

### 6. Analytics Dashboard

Route: `/analytics` accessible from sidebar

Layout:
1. **Productivity Heatmap** (full width): GitHub-style 52-week grid, colored by tasks completed per day. Green gradient: 0→1→3→5→8+ tasks
2. **Completion Rate by Board**: horizontal bar chart (recharts) with percentage label
3. **Tasks by Priority Over Time**: stacked area chart, last 30 days
4. **Focus Time This Week**: bar chart (hours per day)
5. **Average Completion Time** vs Estimated (scatter plot, one dot per task)
6. **Streaks Card**: current habit streaks ranked
7. **Overdue Rate**: donut chart (on-time vs late vs incomplete)

---

### 7. Smart "Today" & "Upcoming" Views

Sidebar navigation items that aggregate across all boards:

**Today view:**
- Pinned tasks (any board)
- Tasks due today
- In-progress tasks
- Overdue tasks (with "Reschedule" quick action)
- Habits due today (with checkboxes inline)
- Running Pomodoro session shown at top if active

**Upcoming view:**
- Grouped by day for the next 14 days
- Each day group is collapsible
- "No tasks" days shown in muted style (still visible for awareness)

---

### 8. Design System & Visual Language

#### Color Palette
- **Background (dark mode default):** `#0A0A0F` (near-black with blue tint)
- **Surface:** `#12121A` cards, `#1A1A26` sidebar
- **Border:** `#2A2A3E` 
- **Accent primary:** `#6366F1` (Indigo-500) — primary actions, active states
- **Accent secondary:** `#8B5CF6` (Violet-500) — secondary elements, gradients
- **Success:** `#10B981` (Emerald-500)
- **Warning:** `#F59E0B` (Amber-500)
- **Danger:** `#EF4444` (Red-500)
- **Text primary:** `#F0F0FF`
- **Text muted:** `#6B7280`

Light mode: clean whites and light grays, same accent palette

#### Typography
- **Font:** Inter (variable font) for UI; `font-feature-settings: "cv11", "ss01"` for clean numerics
- **Scale:** Use Tailwind's default scale (text-sm for metadata, text-base for body, text-lg for card titles, text-2xl for page headers)

#### Motion Design
- **Card drag:** spring physics via dnd-kit + Framer Motion `AnimatePresence`
- **Drawer open/close:** slide from right, `x: "100%" → 0`, duration 250ms, ease "easeOut"
- **View switch:** crossfade, duration 200ms
- **Toast notifications:** slide in from bottom-right, auto-dismiss with shrink animation
- **Progress bars:** `transition-width 600ms ease-in-out`
- **Confetti on task complete:** if task is marked done from kanban, run a tiny particle burst originating from the card (canvas-confetti, 0.8s)
- All motion: respects `prefers-reduced-motion` — if set, use instant transitions

#### Sidebar
- Width: 260px (collapsible to 64px icon-only mode)
- Sections: Navigation | Your Boards | Habits (collapsed by default) | Settings
- Board items: colored dot + board name + unread count badge
- Drag to reorder boards
- "New Board" button at bottom with `+` icon
- User avatar + name at very bottom with logout dropdown

---

### 9. Keyboard Shortcuts

Implement a global shortcut system (no library needed — custom hook):

| Shortcut         | Action                          |
|------------------|---------------------------------|
| `Cmd/Ctrl + K`   | Open command bar / quick add    |
| `N`              | New task in current view        |
| `B`              | Switch to Board view            |
| `L`              | Switch to List view             |
| `C`              | Switch to Calendar view         |
| `M`              | Switch to Matrix view           |
| `F`              | Enter Focus mode                |
| `Escape`         | Close drawer / modal            |
| `Cmd + Z`        | Undo last action                |
| `?`              | Show shortcut cheat sheet modal |

Show a keyboard shortcut tooltip on hover for all interactive elements (300ms delay, not on touch).

---

### 10. Progressive Web App (PWA)

- `next-pwa` or custom service worker via `next.config.js`
- Manifest: name, short_name, icons (512px + 192px), theme_color, display: standalone
- Offline support: cache all static assets + last-fetched task data in IndexedDB
- "Add to Home Screen" prompt on mobile (custom prompt, not browser default)
- Push notifications for reminders (Web Push API, backend sends via `web-push` library)

---

## Auth Flow

- **Register / Login pages:** Centered card on a gradient background (indigo→violet)
- Form validation with Zod schemas, errors shown inline below each field
- JWT stored in `httpOnly` cookie (not localStorage — XSS safe)
- Refresh token rotation: 7-day refresh token, 15-min access token
- Protected routes: Next.js middleware (`middleware.ts`) checks for valid cookie, redirects to `/login` if not present
- **OAuth (optional stretch):** Google OAuth via Spring Security OAuth2 client

---

## CI/CD Pipeline

### `.github/workflows/ci.yml`
Runs on every PR:
1. **Frontend lint:** `npx eslint . --max-warnings 0`
2. **Frontend type check:** `npx tsc --noEmit`
3. **Frontend tests:** `npx jest --ci`
4. **Backend tests:** `./gradlew test` (runs Testcontainers integration tests against a real Postgres)
5. **Backend build:** `./gradlew bootJar`

### `.github/workflows/deploy.yml`
Runs on push to `main`:
1. **Frontend:** Vercel CLI deploy (production) — `vercel --prod`
2. **Backend:** Build Docker image → push to Fly.io registry → `fly deploy --remote-only`
3. **Post-deploy:** smoke test (curl health endpoint, assert 200)

---

## Docker Configuration (Backend)

```dockerfile
# apps/backend/Dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY gradlew settings.gradle.kts build.gradle.kts ./
COPY gradle/ gradle/
RUN ./gradlew dependencies --no-daemon
COPY src/ src/
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

`fly.toml`:
```toml
app = "flux-backend"
primary_region = "iad"

[build]

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = false    # Keep always-on on free tier
  auto_start_machines = true
  min_machines_running = 1

[[vm]]
  memory = "512mb"
  cpu_kind = "shared"
  cpus = 1
```

---

## Environment Variables

### Frontend (Vercel)
```
NEXT_PUBLIC_API_URL=https://flux-backend.fly.dev
NEXT_PUBLIC_WS_URL=wss://flux-backend.fly.dev/ws
NEXT_PUBLIC_APP_URL=https://flux.vercel.app
```

### Backend (Fly.io secrets)
```
DATABASE_URL=postgresql://...  (Supabase connection string with PgBouncer)
JWT_SECRET=<RS256 private key>
JWT_PUBLIC_KEY=<RS256 public key>
ALLOWED_ORIGINS=https://flux.vercel.app
```

---

## Stretch Goals (Do These If Time Allows)

In priority order:

1. **Dark / Light / System theme toggle** — top-right of navbar
2. **Board templates:** create a new board from a template (Software Sprint, Personal Goals, Content Calendar, etc.)
3. **Task relationships:** "blocks / blocked by" links between tasks
4. **Time blocking:** drag a task from the list into a weekly calendar to "schedule" it
5. **Recurring tasks:** RRULE parsing — when a recurring task is completed, auto-generate the next occurrence
6. **Real-time multi-device sync:** WebSocket pushes task mutations to all connected clients for the same user
7. **Search:** `Cmd+F` full-text search across all tasks and descriptions (Postgres `tsvector` full-text index)
8. **Mobile responsive layout:** sidebar collapses to bottom tab bar on screens < 768px
9. **Export:** download all tasks as CSV or JSON
10. **Webhooks / integrations:** configurable outbound webhooks on task events (foundation for Zapier/n8n integration)

---

## Code Quality Requirements

- **No `any` in TypeScript** — use `unknown` + type guards where truly necessary
- **All API calls go through a typed API client** (e.g., `lib/api.ts` with typed request/response functions — no raw `fetch` scattered across components)
- **All Kotlin DTOs are data classes** — no raw `Map<String, Any>`
- **All database queries via Spring Data repositories** — no raw JDBC unless for performance-critical bulk ops
- **Error boundaries in React** — global error boundary + per-route boundaries
- **Loading states everywhere** — skeleton loaders (not spinners) using shadcn `Skeleton` component
- **Optimistic updates** — all mutations (create/move/delete/update) update the UI immediately before the server confirms, then rollback on error
- **CORS locked down** — backend only accepts requests from the frontend origin
- **All endpoints authenticated** except `/api/auth/register` and `/api/auth/login`
- **Input validation:** Zod on frontend, `@Valid` + Bean Validation annotations on backend DTOs

---

## Deliverables Checklist

When done, the following must all be true:

- [ ] App is deployed and accessible at a public URL (Vercel)
- [ ] Backend API is deployed and accessible at a public URL (Fly.io)
- [ ] User can register, log in, and stay logged in across page refreshes
- [ ] User can create boards, add columns, add tasks
- [ ] Drag and drop works: task between columns, task within column, column reorder
- [ ] All 4 views (Kanban, List, Calendar, Matrix) work and show the same data
- [ ] Natural language quick add works (Cmd+K)
- [ ] Focus / Pomodoro mode works with a timer
- [ ] Habit tracker: create habit, mark done, see streak
- [ ] Analytics page renders all charts with real data
- [ ] GitHub Actions CI passes on every PR
- [ ] GitHub Actions CD deploys on every merge to `main`
- [ ] Lighthouse score: Performance ≥ 85, Accessibility ≥ 90, Best Practices ≥ 90
- [ ] All TypeScript strict errors resolved
- [ ] All backend tests pass (`./gradlew test`)
- [ ] README documents: local dev setup, env vars, architecture overview, deployment steps

---

## Notes for the Agent

- **Start with the monorepo scaffold** — get both apps running locally before building features
- **Use Supabase's transaction pooler URL** (port 6543) for JPA — avoids connection limit issues on free tier
- **dnd-kit position strategy:** Use `rectSortingStrategy` for grids and `verticalListSortingStrategy` for lists. For Kanban (cross-container drag), use `closestCorners` collision detection
- **LexoRank for ordering:** Use float positions (e.g., 1000, 2000, 3000) with midpoint insertion (`(a + b) / 2`). Rebalance when gap < 1.0
- **STOMP over WebSocket in Next.js:** Use `@stomp/stompjs` client with `Client` class — do not use SockJS
- **Framer Motion + dnd-kit:** Use `layoutId` for cards so they animate to their new position after a drop. Wrap the sortable list in `AnimatePresence`
- **Confetti:** `canvas-confetti` package — fire from the card's bounding rect on `onDragEnd` when status changes to "done"
- **Pomodoro timer:** Use `useRef` for the interval ID (not state) to avoid re-render loops. Persist timer state in Zustand so navigating away doesn't reset it
