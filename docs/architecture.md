# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

- Presentation tier decision: Keep React responsible for rendering and user interaction. Use the shared API client for relative `/api` requests and give each capability clear loading, empty, and error states.
- Logic tier decision: Keep HTTP parsing, validation, status codes, and response shapes in Express routes; put reusable business behavior in explicit service functions. Return deliberate 4xx responses for invalid input or missing records.
- Data tier decision: Use Mongoose schemas for required fields, types, and defaults. Keep persistence in models and the existing database configuration; preserve idempotent seed upserts and unrelated records.
- Activity tracking approach: Validate request fields and well-formed, existing user and team IDs before creating an activity. Store those references and the submitted type/minutes, compute points in the service, and set `occurredAt` when recording the activity.
- Team management approach: Validate that a name is present, persist the submitted name and mascot, and support member references. Keep list and create responses consistent with the API contract.
- Leaderboard approach: Aggregate activity points by team, order totals descending, and assign 1-based ranks in the service/API response.
- Workout suggestions approach: Find the requested user by email, use their goal and recent activities to select and explain matching workouts, and return a deliberate 4xx when no user matches.

## Test plan

- Data tier — `npm run test:data --prefix octofit-tracker/backend`: **Run; 4 failed, 1 passed.** Failures indicate missing required schema validation for users, teams, activities, and workouts. The seed idempotency/preservation test passed.
- API — `npm run test:api --prefix octofit-tracker/backend`: **Run; 4 failed.** Failures indicate invalid activity/team requests return `501` instead of 4xx, leaderboard results are empty, and workout recommendations do not match the user's goal/activity.
- Presentation — `npm test --prefix octofit-tracker/frontend`: **Run; 15 failed.** Failures indicate missing API data rendering, user-specific workout requests, and accessible loading, empty, and error states across the capability routes.
- Backend build — `npm run build --prefix octofit-tracker/backend`: **Not run yet.** Run after implementation; a failure indicates a backend TypeScript/type-check issue.
- Frontend build — `npm run build --prefix octofit-tracker/frontend`: **Not run yet.** Run after implementation; a failure indicates a frontend production-build or type-check issue.
