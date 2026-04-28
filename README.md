# VisAI — AI-Powered Resume Builder

> Generate ATS-optimized, tailored resumes from job descriptions in seconds.

![Stack](https://img.shields.io/badge/Frontend-React%20%2B%20Vite-61DAFB?style=flat-square&logo=react)
![Stack](https://img.shields.io/badge/Backend-FastAPI%20%2B%20MongoDB-009688?style=flat-square&logo=fastapi)
![Auth](https://img.shields.io/badge/Auth-JWT%20Bearer-orange?style=flat-square)
![Theme](https://img.shields.io/badge/Theme-Dark%20%2F%20Light-blueviolet?style=flat-square)

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Project Structure](#project-structure)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Frontend Setup](#frontend-setup)
  - [Backend Setup](#backend-setup)
  - [Environment Variables](#environment-variables)
- [Authentication](#authentication)
  - [Forgot Password Flow](#forgot-password-flow)
- [API Endpoints](#api-endpoints)
- [Frontend Architecture](#frontend-architecture)
- [Resume Templates](#resume-templates)
- [ATS Scoring](#ats-scoring)
- [Responsive Design](#responsive-design)
- [Contributing](#contributing)

---

## Overview

**VisAI** is a full-stack AI resume builder that lets users:

1. Paste any job description
2. AI rewrites and tailors their resume to match that JD
3. Get an ATS score with matched/missing skills breakdown
4. Export or save multiple tailored resumes per job

---

## Features

- **AI Resume Generation** — Sends your base resume + job description to an LLM; gets back a fully tailored resume
- **ATS Scoring** — Keyword extraction and skill matching against the job description
- **Multiple Templates** — Classic, Modern, Compact, and more via `Templatepicker.jsx`
- **Job Queue** — Add, search, and manage multiple job listings
- **Saved Resumes** — MongoDB-backed save/edit/delete for all generated resumes
- **Base Resume Upload** — Parse your existing resume (PDF/text) and store it as your base profile
- **Profile Management** — Edit name, email, username; change password inline
- **Security Questions** — Used for password recovery without email (set during signup, answer to reset)
- **Dark / Light Theme** — Persisted via `localStorage`
- **Responsive UI** — Works on mobile, tablet, and desktop with hamburger nav
- **JWT Auth** — Token stored in `localStorage`, auto-attached via Axios interceptor; 401 triggers auto-logout

---

## Project Structure

```
├── frontend/
│   ├── src/
│   │   ├── App.jsx                  # Main shell, routing, nav, theme toggle
│   │   ├── App.css                  # Global design tokens + responsive breakpoints
│   │   ├── index.css                # Body/root resets
│   │   ├── main.jsx                 # React root, wraps <AuthProvider>
│   │   ├── Api.jsx                  # Axios instance with JWT interceptor
│   │   ├── LoginPage.jsx            # Login, Signup, Forgot Password, AuthContext
│   │   ├── JobsTab.jsx              # Add jobs, job queue, job cards
│   │   ├── ResumeScreen.jsx         # AI resume generation, editor, ATS score
│   │   ├── Templatepicker.jsx       # Resume template selector (TEMPLATES export)
│   │   ├── SavedTab.jsx             # Saved resumes list + detail view
│   │   ├── ProfileTab.jsx           # User profile editor
│   │   ├── Baseresumetab.jsx        # Base resume upload + parse
│   │   ├── resumeExportUtils.jsx    # PDF/print export helpers
│   │   └── hooks/
│   │       └── useTheme.jsx         # Dark/light theme hook (localStorage)
│   ├── index.html
│   ├── vite.config.js
│   └── package.json
│
└── backend/
    └── main.py                      # FastAPI app (all routes)
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite |
| Styling | CSS Variables, custom design system (no Tailwind) |
| HTTP Client | Axios with request/response interceptors |
| Backend | FastAPI (Python) |
| Database | MongoDB (via Motor async driver) |
| Auth | JWT Bearer tokens (python-jose) |
| AI | OpenAI / Anthropic API (configured server-side) |
| Password Hashing | bcrypt |

---

## Getting Started

### Prerequisites

- Node.js ≥ 18
- Python ≥ 3.10
- MongoDB (local or Atlas)
- An OpenAI or Anthropic API key

---

### Frontend Setup

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/visai.git
cd visai/frontend

# Install dependencies
npm install

# Create env file
cp .env.example .env
# → Set VITE_API_URL (see Environment Variables)

# Start dev server
npm run dev
```

---

### Backend Setup

```bash
cd visai/backend

# Create virtual environment
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate

# Install dependencies
pip install fastapi uvicorn motor pymongo python-jose[cryptography] bcrypt openai anthropic python-multipart

# Create env file
cp .env.example .env
# → Fill in MONGO_URI, SECRET_KEY, OPENAI_API_KEY etc.

# Run server
uvicorn main:app --reload --port 8000
```

---

### Environment Variables

**Frontend** (`.env` in `frontend/`):

```env
VITE_API_URL=http://localhost:8000
```

**Backend** (`.env` in `backend/`):

```env
MONGO_URI=mongodb://localhost:27017
DB_NAME=visai
SECRET_KEY=your-super-secret-jwt-key-here
ACCESS_TOKEN_EXPIRE_MINUTES=10080
OPENAI_API_KEY=sk-...
# or
ANTHROPIC_API_KEY=sk-ant-...
```

---

## Authentication

Auth is handled entirely in `LoginPage.jsx` via the exported `AuthContext`.

### Flow

```
Login / Signup
     ↓
POST /auth/login  or  POST /auth/register
     ↓
JWT token stored in localStorage ("resumeai_token")
     ↓
Axios interceptor attaches token to every request
     ↓
401 response → auto clears token → reloads (forces re-login)
```

### AuthProvider

Wrap your app once in `main.jsx`:

```jsx
import { AuthProvider } from "./LoginPage";

<AuthProvider>
  <App />
</AuthProvider>
```

### useAuth hook

```jsx
const { user, login, logout, refreshUser, loading } = useAuth();
```

| Property | Description |
|---|---|
| `user` | Current user object (`null` if not logged in) |
| `login(email, password)` | Calls `/auth/login`, stores token, sets user |
| `logout()` | Calls `/auth/logout`, clears token + user |
| `refreshUser()` | Re-fetches `/auth/me` and updates user state |
| `loading` | `true` while session is being restored on mount |

---

### Forgot Password Flow

Uses security questions set during signup. No email required.

**During Signup** — user picks 2 security questions and answers. Stored as hashed answers in MongoDB alongside the user document.

**During Forgot Password** (3-step modal on the Login page):

```
Step 1 → Enter email
         POST /auth/forgot-password/questions
         → Returns the user's question prompts (not answers)

Step 2 → Answer both questions
         POST /auth/forgot-password/verify
         → Returns a short-lived reset_token (expires in 15 min)

Step 3 → Set new password
         POST /auth/forgot-password/reset
         → Updates password, invalidates token
```

**Security questions list** (user picks 2 during signup):

- What was the name of your first pet?
- What city were you born in?
- What was your childhood nickname?
- What is your mother's maiden name?
- What was the name of your first school?
- What was your first car?
- What is your oldest sibling's first name?
- What street did you grow up on?
- What was the name of your best friend in childhood?
- What was your favorite sports team growing up?

> **Backend note:** Store answers hashed with bcrypt. Never return raw answers. The `reset_token` should be a short UUID stored in the user document with an `expires_at` timestamp.

---

## API Endpoints

### Auth

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/register` | Create account (accepts `security_questions` array) |
| POST | `/auth/login` | Login → returns `{token, user}` |
| POST | `/auth/logout` | Invalidate session (optional server-side) |
| GET | `/auth/me` | Get current user from token |
| PUT | `/auth/profile` | Update name, email, username, or password |
| POST | `/auth/forgot-password/questions` | Get user's question prompts by email |
| POST | `/auth/forgot-password/verify` | Verify answers → returns reset token |
| POST | `/auth/forgot-password/reset` | Set new password with reset token |

### Jobs

| Method | Endpoint | Description |
|---|---|---|
| GET | `/jobs` | List all jobs for current user |
| POST | `/add-job` | Add a new job (`title`, `description`, `company`) |
| DELETE | `/jobs/{id}` | Delete a job |

### Resumes

| Method | Endpoint | Description |
|---|---|---|
| POST | `/generate-resume` | AI generate tailored resume for a job |
| POST | `/save-resume` | Save a generated resume to MongoDB |
| GET | `/saved-resumes` | List all saved resumes |
| PUT | `/update-resume/{id}` | Update an existing saved resume |
| DELETE | `/delete-resume/{id}` | Delete a saved resume |

### Profile / Base Resume

| Method | Endpoint | Description |
|---|---|---|
| GET | `/profile` | Get stored profile data |
| PUT | `/profile` | Save profile fields |
| POST | `/parse-resume` | Parse uploaded resume text → structured profile |

---

## Frontend Architecture

### State Management

No Redux. State lives in component trees and is passed via props.

| State | Lives in | Passed to |
|---|---|---|
| `user`, `token` | `AuthContext` (LoginPage.jsx) | Whole app via `useAuth()` |
| `view` | `AppShell` (App.jsx) | Controls which tab renders |
| `savedResumes` | `AppShell` | `SavedTab`, `UserDropdown` |
| `profile` | `AppShell` | `ProfileTab`, `ResumeScreen` |
| `baseResume` | `AppShell` | `BaseResumeTab`, `ResumeScreen` |
| `selectedJob` | `AppShell` | `ResumeScreen` |

### Navigation

Desktop: tab bar in the sticky header.
Mobile/Tablet (≤ 900px): hamburger button → slide-down drawer with the same tabs.

```jsx
const NAV = [
  { id: "jobs",       label: "Jobs",        icon: "💼" },
  { id: "saved",      label: "Saved",       icon: "📄", badge: savedResumes.length },
  { id: "profile",    label: "Profile",     icon: "👤" },
  { id: "baseresume", label: "Base Resume", icon: "📝" },
];
```

---

## Resume Templates

Templates are defined and exported from `Templatepicker.jsx`:

```js
export const TEMPLATES = [
  { id: "classic",  name: "Classic",  font: "serif" },
  { id: "modern",   name: "Modern",   font: "sans"  },
  { id: "compact",  name: "Compact",  font: "mono"  },
  // add more here
];
```

`PrintableResume` component (in `ResumeScreen.jsx`) accepts:

```jsx
<PrintableResume
  data={resumeData}
  templateId="classic"
  maxExp={3}
  maxBullet={4}
  font="serif"
  pages={1}
/>
```

---

## ATS Scoring

ATS scoring is done server-side in `/generate-resume`. The response includes:

```json
{
  "resumeData": { ... },
  "scoring": {
    "ats": { "score": 87 },
    "matched_skills": ["Python", "FastAPI", "Docker"],
    "missing_skills": ["Kubernetes", "Terraform"]
  }
}
```

Score color thresholds in the UI:

| Score | Color |
|---|---|
| ≥ 85 | `var(--accent)` (green) |
| ≥ 70 | `var(--warn)` (yellow) |
| < 70 | `var(--danger)` (red) |

---

## Responsive Design

Breakpoints in `App.css`:

| Breakpoint | Behavior |
|---|---|
| > 900px | Full desktop layout, tab bar visible in header |
| ≤ 900px | Hamburger menu, single-column job/saved layouts |
| ≤ 600px | Tighter padding, smaller typography, full-width buttons |
| ≤ 380px | Minimum viable layout for very small phones |

CSS custom property `--page-px` scales the main content padding automatically.

---

## Contributing

1. Fork the repo
2. Create a branch: `git checkout -b feature/your-feature`
3. Make your changes
4. Commit: `git commit -m "feat: describe your change"`
5. Push: `git push origin feature/your-feature`
6. Open a Pull Request

### Commit Convention

```
feat:     new feature
fix:      bug fix
style:    CSS / UI only changes
refactor: code restructure, no behavior change
docs:     README or comments only
chore:    build, deps, config
```

---

## License

MIT — use freely, attribution appreciated.

---

*Built with ❤️ using React + FastAPI + MongoDB*
