# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A full-stack Trello clone: Django REST Framework backend + React frontend. No Docker, no WebSockets — all communication is polling-based REST over HTTP.

## Running the Project

**Redis** (required before starting the backend — must run on port 6380, not the default 6379):
```bash
redis-server --port 6380
```

**Backend** (`/backend`, Python 3.8, pipenv):
```bash
cd backend
pipenv install
pipenv run python manage.py migrate
pipenv run python manage.py runserver   # http://localhost:8000
```

**Frontend** (`/frontend`, Node.js + Yarn):
```bash
cd frontend
yarn install
yarn start   # http://localhost:3000
```

## Running Tests

**Backend** (pytest with coverage, uses `--nomigrations` so runs against a fresh schema):
```bash
cd backend
pipenv run pytest
# HTML coverage report → backend/htmlcov/
```

Run a single test file:
```bash
pipenv run pytest boards/tests/test_views.py
```

**Frontend** (Jest via react-scripts, no test files currently exist):
```bash
cd frontend
yarn test
```

## Architecture

### Backend (`/backend`)

Three Django apps:
- **`users`** — custom `User` model (extends `AbstractUser`), JWT registration/auth
- **`projects`** — teams/projects, membership roles (MEMBER=1, ADMIN=2), invite system
- **`boards`** — boards, lists, items/cards, labels, comments, attachments, notifications

Root URL config: `trello/urls.py`. Board and project endpoints are in `boards/urls.py` and `projects/urls.py`.

**Auth**: JWT via `djangorestframework-simplejwt`. Access token: 1 day, refresh: 360 days. All endpoints require `IsAuthenticated` by default. Public: `register/`, `token/`, `token/refresh/`. Both username and email are accepted at login (two `AUTHENTICATION_BACKENDS`).

**Database**: SQLite (default). Board `owner` is a Generic FK — it can be either a `User` or a `Project`, which is the key polymorphism in the data model.

**Ordering (drag-and-drop)**: `List.order` and `Item.order` use fractional indexing — `DecimalField(max_digits=30, decimal_places=15)`. Default: 65535; inserts use midpoint or `last + 65535`. Frontend updates optimistically, then PUTs the new `order` value.

**Redis** (direct `redis.Redis` client, not Django cache):
- Recently viewed boards: sorted set `{username}:RecentlyViewedBoards`
- Invite tokens: hash `ProjectInvitation:{uuid4}` — one-time use, deleted on acceptance
- Client is instantiated at module level in `boards/views.py` and `projects/views.py` — **startup fails if Redis is not running**.

**Signals** (`boards/signals.py`, `projects/signals.py`):
- `Project` post_save → creates owner's `ProjectMembership` as ADMIN
- `Board` post_save → creates 10 default `Label` objects
- `Comment` post_save/post_delete → creates/removes `Notification` for assigned users

**Email**: `EMAIL_BACKEND = locmem` — project invite emails are never actually delivered.

### Frontend (`/frontend`)

React 16.13.1 (Create React App). No Redux — state is `useContext` + `useReducer`.

**Global state** (`src/context/`): `{ authUser, checkedAuth, board, setBoard }`. JWTs stored in `localStorage` (`accessToken`, `refreshToken`). Auth check on mount via `GET /me/`; app renders nothing until `checkedAuth = true`.

**HTTP client** (`src/static/js/util.js`): `authAxios` — Axios instance that injects `Authorization: Bearer` header. Response interceptor auto-retries once on 401 by refreshing the token; if refresh fails, the promise rejects (user must log in again).

**Key constant**: backend URL is hardcoded in `src/static/js/const.js`:
```js
export const backendUrl = "http://localhost:8000";
```

**Routing** (`src/App.js`): `/` (Home), `/b/:id` (Board), `/p/:id` (Project) when authenticated; Landing/Login/Register when not.

**Drag-and-drop**: `react-beautiful-dnd`. All board mutation logic (drag-end, add/update/remove list/card) lives in `src/static/js/board.js`.

**Custom hooks** (`src/hooks/`): `useAxiosGet` (fetch on mount + helpers), `useBlurSetState` (close-on-outside-click), `useDocumentTitle`.

**SCSS**: Source files in `src/static/css/`. The compiled `index.css` is committed and is what CRA uses — SASS is not compiled automatically (no sass package in `package.json`). Edit `index.css` directly or compile SCSS manually.

## Known Issues / Gotchas

- `LabelDetail` is defined twice in `boards/views.py` — the second definition (line ~416) with `permission_classes = [AllowAny]` shadows the first, making label edit/delete unauthenticated. This is a bug.
- `AttachmentList` and `AttachmentDetail` also use `AllowAny` — attachments are publicly accessible without auth.
- `SECRET_KEY` is hardcoded in `settings.py` — no `.env` file or environment variable management.

## Optional: Unsplash Board Backgrounds

Set `REACT_APP_UNSPLASH_API_ACCESS_KEY` in a `.env` file inside `/frontend/` to enable Unsplash image search for board backgrounds.
