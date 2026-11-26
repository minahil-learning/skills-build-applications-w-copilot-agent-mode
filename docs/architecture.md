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

- Presentation tier decision: Not implemented in the data-tier change; React routes and components remain outside this tier and will consume `/api/*` responses through the shared API client.
- Logic tier decision: Not implemented in the data-tier change. Express routes and `octofitService` remain responsible for request validation, HTTP status decisions, and cross-model operations in a later tier.
- Data tier decision: Mongoose owns schema validation and relationships. `User` requires `name`, `email`, and `fitnessGoal`; `email` is trimmed and lowercased. `Team` requires `name` and `mascot`, with `members` defaulting to an empty array and referencing `User`. `Activity` requires `user`, `team`, `type`, `minutes`, `points`, and `occurredAt`; its user/team fields reference `User` and `Team`, minutes must be at least 1, and points must be at least 0. `Workout` requires `title`, `focus`, `minutes`, and `difficulty`, with minutes required to be at least 1.
- Activity tracking approach: Store required user/team ObjectId relationships, activity type, duration, computed points, and occurrence date. Data-tier validation prevents malformed or incomplete activity documents; lookup of existing referenced records and deliberate HTTP 4xx responses remain the logic-tier responsibility.
- Team management approach: Require a team name and mascot, default members to an empty array, and store member ObjectIds with a `User` reference. Seed membership is reconciled explicitly when a demo user's team changes.
- Leaderboard approach: The data model provides team relationships and activity points so later logic can aggregate `Activity.points` by team and assign ranks.
- Workout suggestions approach: The data model provides workout title, focus, duration, and difficulty. Later logic can match a user's fitness goal or recent activity type to a workout focus and explain the match.

## Test plan

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`
  - Verifies required Mongoose fields and relationships, team member defaults, seed data, and idempotent reruns.
  - Result: 4 tests passed. The representative seeding test remained blocked by MongoDB Memory Server downloading MongoDB `7.0.24` and exceeded the suite's fixed 60-second timeout. This is a binary-download timeout, not a reported assertion failure in the seed implementation.
- API command: `npm run test:api --prefix octofit-tracker/backend`
  - Verifies activity, team, leaderboard, and workout API contracts once the logic tier is implemented.
  - Earlier verification result: 4 API tests were skipped because MongoDB Memory Server startup timed out at 60 seconds while preparing MongoDB.
- Frontend command: `npm test --prefix octofit-tracker/frontend`
  - Verifies routing, API rendering, loading, empty, and accessible failure states once the presentation tier is implemented.
  - Earlier verification result: 15 starter presentation tests failed because the scaffold did not yet render or request the required data and states.
- Backend build command: `npm run build --prefix octofit-tracker/backend`
  - Verifies that the changed Mongoose and seed TypeScript compiles.
  - Result: passed (`tsc --noEmit`).
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`
  - Verifies the frontend tier build once presentation work is completed.

## Seed-data strategy

- Demo teams, users, activities, and workouts are created with stable upserts rather than collection deletion.
- Demo users are keyed by email and update name, team, and fitness goal on rerun.
- If a demo user is moved to a different team, the previous team's `members` array loses that user's ObjectId and the new team's array gains it with `$addToSet`.
- Activities and workouts are keyed by stable demo fields and updated with their seeded values, so reruns do not duplicate representative records.
- `seed.ts` does not clear `User`, `Team`, `Activity`, or `Workout`; sentinel and unrelated records are retained.
