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

- **Presentation tier decision:** Keep page components focused on rendering and user interaction. Use the shared API client for requests, and give each capability consistent accessible loading, empty, and error states.
- **Logic tier decision:** Keep HTTP handling and request validation in Express routes, with capability logic in backend services where appropriate. Return deliberate 4xx responses for invalid inputs and unknown users; do not allow malformed IDs to surface as cast-error 500s. Do not add authentication.
- **Data tier decision:** Use Mongoose schemas to enforce required identity, goal, team, activity, and workout fields and appropriate references/defaults. Keep seeding additive and idempotent so existing unrelated records are preserved.
- **Activity tracking approach:** Validate required fields, ID format, and user/team existence before creating an activity. Compute points on the server using the established scoring rule, assign the occurrence date on creation, and return the persisted activity in the same shape used by the list endpoint.
- **Team management approach:** Create and list teams using the submitted name and mascot, preserve the members array, and return `400` when the required name is absent.
- **Leaderboard approach:** Derive each team's total from its activity points, order totals highest first, and assign ranks starting at 1 in the returned order.
- **Workout suggestions approach:** Find the user by email, prefer matching workouts to the user's fitness goal, and use recent activity as a relevant focus when appropriate. Include a human-readable reason; return a deliberate 4xx for an unknown email.

## Test plan and current results

Run the focused contract tests after each tier change, then run both builds as final checks. Results below reflect the commands run on 2026-10-27.

- **Data-tier command:** `npm run test:data --prefix octofit-tracker/backend`
  - **Result:** 4 failed, 1 passed. Required-field validation is missing for User, Team, Activity, and Workout. The seed/idempotency test passed.
  - **Failure meaning:** The data schemas do not yet enforce the required fields in the data contract.
- **API command:** `npm run test:api --prefix octofit-tracker/backend`
  - **Result:** 4 failed. Invalid activity and team requests returned `501` instead of `400`; the leaderboard returned an empty list; the first workout suggestion did not match the user's goal/activity.
  - **Failure meaning:** API handlers are incomplete or do not yet satisfy validation, aggregation, and recommendation behavior.
- **Frontend command:** `npm test --prefix octofit-tracker/frontend`
  - **Result:** 15 failed. The tests could not find rendered API data or the expected workout-suggestion request, and the required loading, empty, and accessible error states were missing.
  - **Failure meaning:** Presentation components are not yet wired to meet the API rendering and request-state contracts.
- **Backend build command:** `npm run build --prefix octofit-tracker/backend`
  - **Result:** Not run in this pass.
  - **Failure meaning:** A failure would indicate backend TypeScript compilation or type errors.
- **Frontend build command:** `npm run build --prefix octofit-tracker/frontend`
  - **Result:** Not run in this pass.
  - **Failure meaning:** A failure would indicate frontend TypeScript or Vite production-build errors.
