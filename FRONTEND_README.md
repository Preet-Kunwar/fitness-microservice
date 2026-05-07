# Fitness Tracker — React Frontend

A modern single-page application (SPA) built with React 19 and Vite. It connects to the Fitness Microservices backend through an OAuth2 PKCE authentication flow powered by Keycloak, and provides a clean interface for logging workouts and viewing AI-generated fitness recommendations.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Environment & Configuration](#environment--configuration)
- [Project Structure](#project-structure)
- [Authentication Flow](#authentication-flow)
- [State Management](#state-management)
- [Pages & Components](#pages--components)
- [API Integration](#api-integration)
- [Available Scripts](#available-scripts)
- [Known Issues & Improvements](#known-issues--improvements)

---

## Features

- Keycloak OAuth2 PKCE login/logout — no credentials ever stored in the app
- Protected routes — unauthenticated users are shown a login screen
- Log fitness activities (type, duration, calories burned)
- Browse a list of all your activities
- Drill into an activity to read its AI-generated recommendation (analysis, improvements, workout suggestions, safety guidelines)
- JWT and user ID automatically attached to every API request via an Axios interceptor
- Redux Toolkit slice persists auth state to `localStorage` across page refreshes

---

## Tech Stack

| Category | Library / Tool |
|---|---|
| UI Framework | React 19 |
| Build Tool | Vite 6 |
| Component Library | MUI (Material UI) v6 |
| Styling | Emotion (`@emotion/react`, `@emotion/styled`) |
| State Management | Redux Toolkit + React-Redux |
| Routing | React Router v7 |
| HTTP Client | Axios |
| OAuth2 / OIDC | react-oauth2-code-pkce |
| Linting | ESLint 9 with react-hooks + react-refresh plugins |

---

## Prerequisites

- **Node.js** 20 or higher (React Router v7 requires Node ≥ 20)
- **npm** 8 or higher
- The full backend must be running (see the backend README)
- Keycloak running on `http://localhost:8181` with realm `fitness-oauth2` and client `oauth2-pkce-client` configured

---

## Getting Started

```bash
# Clone the repo (if not already done)
git clone <repo-url>
cd fitness-app-frontend

# Install dependencies
npm install

# Start the development server
npm run dev
```

The app will be available at **http://localhost:5173**.

---

## Environment & Configuration

There is no `.env` file required for local development. All configuration is hardcoded in two source files that you can edit directly:

### `src/authConfig.js` — Keycloak / OAuth2 settings

```js
export const authConfig = {
  clientId: 'oauth2-pkce-client',
  authorizationEndpoint: 'http://localhost:8181/realms/fitness-oauth2/protocol/openid-connect/auth',
  tokenEndpoint:         'http://localhost:8181/realms/fitness-oauth2/protocol/openid-connect/token',
  redirectUri:           'http://localhost:5173',
  scope: 'openid profile email offline_access',
  onRefreshTokenExpire: (event) => event.logIn(),
}
```

| Property | Purpose |
|---|---|
| `clientId` | Must match the Keycloak client ID exactly |
| `authorizationEndpoint` | Keycloak auth endpoint for the realm |
| `tokenEndpoint` | Keycloak token endpoint |
| `redirectUri` | Must be listed in Keycloak's "Valid redirect URIs" |
| `scope` | Requested OAuth2 scopes; `offline_access` enables refresh tokens |
| `onRefreshTokenExpire` | Automatically redirects to login when the refresh token expires |

### `src/services/api.js` — Backend base URL

```js
const API_URL = 'http://localhost:8080/api';
```

Change this if your gateway runs on a different host or port.

---

## Project Structure

```
fitness-app-frontend/
├── public/
│   └── vite.svg
├── src/
│   ├── assets/
│   │   └── react.svg
│   ├── components/
│   │   ├── ActivityDetail.jsx   # Activity + AI recommendation view
│   │   ├── ActivityForm.jsx     # Form to log a new activity
│   │   └── ActivityList.jsx     # Card grid of past activities
│   ├── services/
│   │   └── api.js               # Axios instance + all API calls
│   ├── store/
│   │   ├── authSlice.js         # Redux slice — auth state + localStorage sync
│   │   └── store.js             # Redux store configuration
│   ├── App.jsx                  # Root component — routing + auth gate
│   ├── App.css
│   ├── authConfig.js            # Keycloak PKCE configuration
│   ├── index.css
│   └── main.jsx                 # React root — AuthProvider + Redux Provider
├── index.html
├── package.json
├── vite.config.js
└── eslint.config.js
```

---

## Authentication Flow

The app uses the **Authorization Code + PKCE** flow — the most secure option for SPAs because no client secret is required.

```
1. User clicks LOGIN
        │
        ▼
2. react-oauth2-code-pkce generates a PKCE code_verifier + code_challenge
        │
        ▼
3. Browser redirects to Keycloak login page
        │
        ▼
4. User enters credentials in Keycloak
        │
        ▼
5. Keycloak redirects back to http://localhost:5173?code=...
        │
        ▼
6. Library exchanges the code for access_token + refresh_token
        │
        ▼
7. App.jsx useEffect fires → dispatches setCredentials to Redux
   (token + userId persisted to localStorage)
        │
        ▼
8. User is routed to /activities
```

On every subsequent page load, Redux initialises from `localStorage` so the user stays logged in until the refresh token expires.

**Logout** calls Keycloak's end-session endpoint and clears localStorage via the `logout` Redux action.

---

## State Management

Redux Toolkit manages a single `auth` slice:

```js
// Initial state (hydrated from localStorage on load)
{
  user:   { sub, email, given_name, ... },  // decoded JWT claims
  token:  "eyJhbGci...",                    // raw access token
  userId: "keycloak-sub-uuid"               // used as X-User-ID header
}
```

### Actions

| Action | Effect |
|---|---|
| `setCredentials({ token, user })` | Saves token + user to state and localStorage |
| `logout()` | Clears state and removes all auth keys from localStorage |

The Axios interceptor in `api.js` reads `token` and `userId` directly from `localStorage` rather than from the Redux store — this keeps the service layer decoupled from React.

---

## Pages & Components

### `App.jsx` — Root / Auth Gate

Renders a full-page login prompt when no token is present. Once authenticated, wraps all routes in a `Box` with a Logout button.

Routes:
- `/` → redirects to `/activities` if authenticated
- `/activities` → `ActivitiesPage` (form + list)
- `/activities/:id` → `ActivityDetail`

### `ActivityForm.jsx`

A controlled form with three fields:

| Field | Type | Options |
|---|---|---|
| Activity Type | Select | RUNNING, WALKING, CYCLING |
| Duration | Number input | minutes |
| Calories Burned | Number input | kcal |

On submit, calls `addActivity(activity)` then invokes `onActivityAdded()` to trigger a page reload (refreshing the activity list).

### `ActivityList.jsx`

Fetches all activities for the authenticated user on mount and renders them as a MUI `Card` grid. Each card is clickable and navigates to `/activities/:id`.

### `ActivityDetail.jsx`

Fetches the full recommendation object for the given activity ID from `/api/recommendations/activity/:id`. Displays two MUI cards side by side:

1. **Activity Details** — type, duration, calories, date
2. **AI Recommendation** — analysis text, bulleted improvements, workout suggestions, safety guidelines

---

## API Integration

`src/services/api.js` exposes three functions, all using a shared Axios instance with base URL `http://localhost:8080/api`:

```js
// Fetch all activities for the current user
export const getActivities = () => api.get('/activities');

// Log a new activity
export const addActivity = (activity) => api.post('/activities', activity);

// Fetch AI recommendation for a specific activity
export const getActivityDetail = (id) => api.get(`/recommendations/activity/${id}`);
```

The request interceptor automatically injects:

```
Authorization: Bearer <token>
X-User-ID: <keycloak-sub>
```

Both values come from `localStorage`. If either is missing (e.g. before login), the header is simply omitted.

---

## Available Scripts

| Command | Description |
|---|---|
| `npm run dev` | Start Vite dev server with HMR at `http://localhost:5173` |
| `npm run build` | Production build output to `dist/` |
| `npm run preview` | Serve the production build locally for testing |
| `npm run lint` | Run ESLint across all `.js` and `.jsx` files |

---

## Known Issues & Improvements

The following items are present in the current codebase and are good candidates for a follow-up:

**Bug — `ActivityDetail.jsx` improvement rendering:**
The improvements list maps over `activity.improvements` but renders `activity.improvements` (the whole array) instead of the individual `improvement` item. Fix:
```jsx
// Current (incorrect)
<Typography key={index} paragraph>• {activity.improvements}</Typography>

// Correct
<Typography key={index} paragraph>• {improvement}</Typography>
```

**`ActivityForm.jsx` prop name mismatch:**
The component receives `onActivitiesAdded` from `App.jsx` but internally calls `onActivityAdded` (no "s"). Both sides should use the same name.

**Hardcoded activity types in `ActivityForm`:**
Only `RUNNING`, `WALKING`, and `CYCLING` are selectable in the form, but the backend supports `SWIMMING`, `WEIGHT_TRAINING`, `YOGA`, `HIIT`, `CARDIO`, `STRETCHING`, and `OTHER` as well.

**No loading or error states:**
API calls in components catch errors with `console.error` but show no user-facing feedback. Adding MUI `Alert` or `Snackbar` components would improve UX significantly.

**`window.location.reload()` after activity submit:**
A cleaner solution is to re-fetch the activity list via the existing `fetchActivities` function rather than forcing a full page reload.

**Production readiness:**
For a production deployment, move `API_URL`, `authorizationEndpoint`, `tokenEndpoint`, and `redirectUri` into `.env` files (`VITE_` prefixed) so they can be configured per environment without code changes.
