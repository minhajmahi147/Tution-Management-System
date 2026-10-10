# Tutor Management Platform

A TutorSheba-style web app that connects students/guardians with home tutors. Students post tuition requests, tutors register with their preferred areas, and both can browse each other's listings.

## Architecture

```
tutor-management-platform/
├── server/              # Express + MongoDB REST API (deployed on Render)
└── tutorsheba-clone/    # React + Vite + Tailwind frontend (deployed on Vercel)
```

```
Browser (React SPA) ──HTTP/JSON──> Express API (/api/auth/*) ──Mongoose──> MongoDB
```

- Production API: `https://tutor-management-platform.onrender.com`
- Auth: passwords hashed with `bcryptjs`; login/register return a JWT (1h expiry) and also store the user in an `express-session` cookie.

## Tech Stack

| Layer    | Tools |
|----------|-------|
| Frontend | React 19, React Router 7, Vite 6, Tailwind CSS 4, Axios, Headless UI, react-select, react-slick |
| Backend  | Node.js, Express 4, Mongoose 8, bcryptjs, jsonwebtoken, express-session, cors, dotenv |
| Database | MongoDB |

## Data Models

### `User` (`server/model/User.js`)

| Field            | Type       | Notes |
|------------------|------------|-------|
| `name`           | String     | required |
| `gender`         | String     | tutors only |
| `email`          | String     | required for tutors |
| `phone`          | String     | required |
| `district`       | String     | required for tutors |
| `location`       | String     | required for tutors |
| `preferredAreas` | [String]   | required (non-empty) for tutors |
| `password`       | String     | required, bcrypt hash |
| `userType`       | String     | `"tutor"` or `"student"` |

### `TutorRequest` (`server/model/tutor_request.js`)

| Field         | Type   | Notes |
|---------------|--------|-------|
| `studentName` | String | required |
| `phone`       | String | required |
| `media`       | String | `Bangla` \| `English` \| `Hindi` |
| `classLevel`  | String | `Class 1` \| `Class 2` \| `Other` |
| `district`    | String | required |
| `area`        | String | required |
| `createdAt`   | Date   | defaults to now |

## API Reference

All routes are mounted under `/api/auth` (`server/routes/auth.js`).

| Method | Endpoint                     | Description |
|--------|------------------------------|-------------|
| POST   | `/register`                  | Register a tutor or student |
| POST   | `/login`                     | Log in with `emailOrPhone`, `password`, `userType` |
| GET    | `/logout`                    | Destroy the session |
| POST   | `/create_tutor_request`      | Create a tuition request (requires `agreeTerms: true`) |
| GET    | `/get_tutor_request`         | List all tuition requests |
| GET    | `/get_tutor_request/:id`     | Get one tuition request |
| GET    | `/tutors`                    | List all tutors (password excluded) |
| GET    | `/tutors/:id`                | Get one tutor |
| GET    | `/get_all_users`             | List all tutors and students |
| GET    | `/get_all_users/:id`         | Get one user |

### Example: register a tutor

```json
POST /api/auth/register
{
  "name": "Rahim",
  "gender": "Male",
  "email": "rahim@example.com",
  "phone": "01700000000",
  "district": "Dhaka",
  "location": "Mirpur",
  "preferredAreas": ["Mirpur", "Uttara"],
  "password": "secret",
  "confirmPassword": "secret",
  "userType": "tutor"
}
```

Students only need `name`, `phone`, `password`, `confirmPassword`, and `userType`.

### Example: login

```json
POST /api/auth/login
{ "emailOrPhone": "01700000000", "password": "secret", "userType": "tutor" }
```

Response: `{ "message": "Login successful", "user": { "id", "email", "userType", "token" } }`

## Frontend Routes

Defined in `tutorsheba-clone/src/App.jsx`:

| Path                          | Page |
|-------------------------------|------|
| `/`                           | Home (hero, services, featured tutors, charts, work process) |
| `/tution_list`                | Browse tuition requests |
| `/tutor_details/:id`          | Tuition request details |
| `/request_for_tutor`          | Post a tuition request |
| `/premium_tutor`              | Browse tutors |
| `/premium_tutor_details/:id`  | Tutor profile |
| `/login`                      | Login |
| `/signup`                     | Register (tutor/student) |
| `/dashboard/:id`              | User dashboard + logout |

`vercel.json` rewrites all paths to `/` so client-side routing works on refresh.

## Getting Started

### Prerequisites

- Node.js 18+
- A MongoDB instance (local or Atlas)

### Backend

```bash
cd server
npm install
```

Create `server/.env`:

```env
MONGO_URI=mongodb+srv://<user>:<pass>@<cluster>/<db>
SESSION_SECRET=change-me
JWT_SECRET=change-me
PORT=5000
```

Run it (there is no `start` script yet):

```bash
node server.js
```

### Frontend

```bash
cd tutorsheba-clone
npm install
npm run dev        # http://localhost:5173
npm run build      # production build to dist/
npm run lint
```

`tutorsheba-clone/.env` defines `VITE_BACKEND_URL`, but most pages currently hardcode the Render URL in their `fetch`/`axios` calls. To develop against a local backend, replace `https://tutor-management-platform.onrender.com` with `http://localhost:5000` in `src/pages/*.jsx`.

## Known Issues

- **Backend URL is hardcoded** across pages instead of using `import.meta.env.VITE_BACKEND_URL`.
- **`src/service/api.jsx` is stale**: it points to `localhost`, calls `/api/tutors` and `/api/auth/signup`, which don't exist (real routes are `/api/auth/tutors` and `/api/auth/register`).
- **JWT secret mismatch**: tokens are signed with `SESSION_SECRET`, but the `authenticateUser` middleware verifies with `JWT_SECRET`. The middleware isn't applied to any route yet, so all endpoints are public.
- **No authorization**: `/get_all_users` exposes every user (minus passwords) to anyone.
- **Session cookies won't cross origins**: `cors()` is used without `credentials`, so the session set by the Render API isn't sent back from the Vercel frontend.
- **Limited `classLevel` enum**: only `Class 1`, `Class 2`, `Other`.
- Frontend `.env` files are committed (they only hold the public API URL); keep `server/.env` out of version control.
