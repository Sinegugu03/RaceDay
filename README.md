# RaceDay – PROG6212 POE

**Student:** Sinegugu Sibanyoni (ST10479060)
**Module:** PROG6212 – Portfolio of Evidence

RaceDay is a race-event management system. Organisers publish events (runs, walks and cycle races) with several race categories (e.g. Under 20, Senior, 10km, 21km). Participants register, enrol in a category, pay their entry fee and view their results once the race has taken place.

| Part | Content | Location |
|---|---|---|
| **Part 1** | System planning and database: ERD, API endpoint plan, SQL Server script, GitHub Actions structure check | [`/docs`](./docs) |
| **Part 2** | RESTful API (ASP.NET Core, EF Core Code-First, session authentication, Swagger, unit tests, CI) | [`/Part2-API`](./Part2-API) |

---

## System roles

**Organiser** – creates, updates and deletes their own events; defines categories for each event; views all enrolments for their events; captures finish times and positions after a race. An Organiser cannot enrol in events and cannot edit another Organiser's events, categories or results.

**Participant** – registers and logs in, views and updates their own profile, browses events and categories, enrols in a category, and views their own enrolments and results. A Participant cannot create or manage events or capture results.

Both roles can view events and categories and view or update their own profile. Anyone can view events, categories and a category's public results leaderboard.

---

# Part 1 – System Planning and Database

## Contents of /docs

| File | Description |
|---|---|
| [`raceday erd.png`](./docs/raceday%20erd.png) | Entity Relationship Diagram – 6 entities with primary keys, foreign keys and cardinality |
| [`raceday endpoint plan.md`](./docs/raceday%20endpoint%20plan.md) | Planned REST API endpoints: authentication, user profile, events, categories, enrolments and results |
| [`sinegugu raceday schema.sql`](./docs/sinegugu%20raceday%20schema.sql) | SQL Server script that creates the full schema and seeds sample data |
| [`Part2_Schema_Changes.sql`](./docs/Part2_Schema_Changes.sql) | Adds `EventType` and `DistanceKm` to the Events table (required by Part 2) |

## Part 1 CI/CD

The workflow [`validate-structure.yml`](./.github/workflows/validate-structure.yml) runs on every push and pull request to `main`. It checks that the `/docs` folder exists and contains the ERD, the endpoint plan and the SQL script, and that a `README.md` is present.

**Successful build screenshot:**

![Part 1 CI/CD green build](./docs/ci-success-screenshot.png)

## Part 1 video walkthrough

**[Watch the Part 1 walkthrough](https://youtu.be/REPLACE_WITH_PART1_VIDEO_ID)**

---

# Part 2 – RESTful API

The API powers the whole RaceDay system. It handles authentication, role-based access, events, categories, enrolments and results, and is documented and testable through Swagger. It follows the Part 1 endpoint plan (differences are listed below).

## Tech stack

- ASP.NET Core 8 Web API (C#)
- Entity Framework Core 8 – Code-First with migrations, SQL Server
- Server-side **session** authentication (user id and role stored in the session) with PBKDF2 password hashing
- Swagger / Swashbuckle with XML-documented endpoints
- xUnit and `WebApplicationFactory` with an in-memory database for unit tests
- GitHub Actions for CI

## Project structure

```
Part2-API/
├── RaceDay.sln
├── RaceDay.Api/
│   ├── Models/        entities (match the Part 1 ERD)
│   ├── Data/          RaceDayDbContext and development seeder
│   ├── Migrations/    EF Core Code-First migrations
│   ├── Dtos/          request/response models and mapper
│   ├── Services/      password hasher and session helpers
│   ├── Filters/       SessionAuthorize (role-based access control)
│   ├── Controllers/   Auth, Users, Events, Categories, Enrolments, Results
│   └── Program.cs
└── RaceDay.Tests/     51 unit tests
```

## Setup and run instructions

**Requirements:** Visual Studio 2022 (ASP.NET workload) or the .NET 8 SDK, and SQL Server (LocalDB, Express or full).

1. **Clone** the repository and open `Part2-API/RaceDay.sln`.
2. **Set the connection string** in `Part2-API/RaceDay.Api/appsettings.json` (`ConnectionStrings:DefaultConnection`) so that `Server=` points to your SQL Server instance (for example `localhost`, `.\SQLEXPRESS` or `(localdb)\MSSQLLocalDB`). The database `RaceDayApiDB` is created automatically.
3. **Create the database** (the API also applies migrations when it starts):
   ```
   cd Part2-API
   dotnet ef database update --project RaceDay.Api
   ```
   or run `Update-Database` in the Visual Studio Package Manager Console.
4. **Run the API:** press F5 in Visual Studio, or
   ```
   dotnet run --project RaceDay.Api --launch-profile http
   ```
5. **Open Swagger:** <http://localhost:5080/swagger> (or <https://localhost:7080/swagger> with the `https` profile).
6. **Demo accounts** (Development environment only, password `Password123!`):
   - `organiser@raceday.test` – Organiser
   - `participant@raceday.test` – Participant
7. **Using sessions in Swagger:** call `POST /api/auth/login` once; the browser keeps the session cookie and Swagger sends it automatically. Use `POST /api/auth/logout` to switch accounts, or use a second Incognito window to act as both roles at the same time.

**Run the unit tests:**
```
cd Part2-API
dotnet test
```

## Authentication

1. `POST /api/auth/register` – creates an account as an **Organiser** or **Participant**. The password is hashed with PBKDF2-SHA256 and a random salt; the plain text is never stored or returned.
2. `POST /api/auth/login` – verifies the credentials and stores the user's id and role in a **server-side session**. The browser holds only a session cookie (30-minute idle timeout).
3. Protected endpoints use a `[SessionAuthorize]` filter: no session returns **401 Unauthorized**; the wrong role returns **403 Forbidden**. Organisers can only change the events, categories and results that belong to them.
4. `POST /api/auth/logout` – clears the session.

## Endpoints (22)

| Area | Method and route | Access |
|---|---|---|
| Auth | `POST /api/auth/register`, `POST /api/auth/login` | Public |
| Auth | `POST /api/auth/logout` | Logged in |
| Profile | `GET /api/users/me`, `PUT /api/users/me` | Logged in |
| Profile | `GET /api/users/{id}` | Organiser |
| Events | `GET /api/events`, `GET /api/events/{id}` | Public |
| Events | `POST /api/events`, `PUT /api/events/{id}`, `DELETE /api/events/{id}` | Organiser (owner) |
| Categories | `GET /api/events/{eventId}/categories` | Public |
| Categories | `POST /api/events/{eventId}/categories`, `PUT /api/categories/{id}`, `DELETE /api/categories/{id}` | Organiser (owner) |
| Enrolments | `POST /api/categories/{categoryId}/enrolments`, `GET /api/users/me/enrolments` | Participant |
| Enrolments | `GET /api/categories/{categoryId}/enrolments` | Organiser (owner) |
| Enrolments | `DELETE /api/enrolments/{id}` | Participant (owner) or Organiser (owner) |
| Results | `POST /api/enrolments/{enrolmentId}/result` | Organiser (owner) |
| Results | `GET /api/enrolments/{enrolmentId}/result` | Participant (owner) or Organiser (owner) |
| Results | `GET /api/categories/{categoryId}/results` | Public |

Every endpoint is visible in Swagger with its purpose, the data it expects and the responses it returns.

## Differences from the Part 1 plan

| Part 1 plan | Part 2 implementation | Reason |
|---|---|---|
| Login returns a JWT `{ token, userId, role }` | Login sets a session cookie and returns `{ userId, fullName, role, message }` | Part 2 requires session-based authentication |
| No logout endpoint | Added `POST /api/auth/logout` | Needed to end a session |
| Events had no type or distance | Added `EventType` (Run, Walk, Cycle) and `DistanceKm` to Events | Required by the Part 2 brief; see `docs/Part2_Schema_Changes.sql` |
| Enrolment default status `Pending` | New enrolments are saved as `Confirmed` | No payment workflow in Part 2 |
| Delete event or category: 204 / 403 / 404 | Also **409** when enrolments exist | Protects participant data |
| Enrol: 201 / 404 / 409 | **409** also when the event is Closed, Completed or Cancelled | Business rule |
| Create event: 201 / 400 | Event dates in the past are rejected (400) | Data quality |
| `Payments` table in the ERD | Entity exists in the model; no endpoints yet | Not required until Part 3 |
| Database `RaceDayDB` (Part 1 script) | API database `RaceDayApiDB` created by EF Core migrations | Keeps the Part 1 script untouched |

## Unit tests

51 xUnit tests start the real API in memory (`WebApplicationFactory`) against an in-memory database, so they need no SQL Server. They cover:

- registration and login (success, validation errors, duplicate email, Admin role rejected, wrong password, hashed passwords, logout)
- event and category management by Organisers
- **role enforcement:** Participants get 403 on Organiser endpoints, users who are not logged in get 401, and Organisers cannot touch other Organisers' events
- enrolments: success, the participant/event/category link being recorded, duplicate, full category and closed event
- results: capture, ownership rules and the public leaderboard

## Part 2 CI/CD

The workflow [`part2-build-test.yml`](./.github/workflows/part2-build-test.yml) restores, builds and runs all unit tests on every push and pull request to `main`.

**Successful build screenshot:**

![Part 2 CI/CD green build](./docs/ci-part2-success-screenshot.png)

<!-- Optional evidence screenshots - delete the comment markers once the images are in /docs

## Evidence screenshots

**Role enforcement – a Participant is blocked from creating an event (403 Forbidden):**

![403 Forbidden](./docs/swagger-403-participant.png)

**Enrolment records the participant, event and category (201):**

![Enrolment](./docs/swagger-201-enrolment.png)

**Passwords are stored hashed, not as plain text:**

![Hashed passwords](./docs/ssms-users-hashed-passwords.png)

-->

## Part 2 video walkthrough

Unlisted YouTube video demonstrating the running API, the code structure, the endpoints, authentication and the unit tests:

**[Watch the Part 2 walkthrough](https://youtu.be/REPLACE_WITH_PART2_VIDEO_ID)**
