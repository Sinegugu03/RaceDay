# RaceDay – PROG6212 POE Part 2: RESTful API

**Student:** Sinegugu Sibanyoni (ST10479060)

RaceDay is a race-event management system. Organisers create events (runs, walks and cycle races) with categories such as Under 20, Senior, 10km and 21km. Participants register, enrol in a category and view their results after the race.

This repository contains the RaceDay Web API: an ASP.NET Core 8 API connected to SQL Server with Entity Framework Core (Code-First), session-based authentication, Swagger, unit tests and a GitHub Actions workflow.

## The two roles

**Organiser** – creates, updates and deletes their own events, adds categories to them, views all enrolments for their events, and captures finish times and positions after a race. An Organiser cannot enrol in events or change another Organiser's events.

**Participant** – registers and logs in, views and updates their own profile, browses events and categories, enrols in a category, and views their own enrolments and results. A Participant cannot create events or capture results.

Both roles can view events and categories and view or update their own profile.

## Tech stack

- ASP.NET Core 8 Web API (C#)
- Entity Framework Core 8 (Code-First, migrations) with SQL Server
- Session-based authentication, passwords hashed with PBKDF2
- Swagger (Swashbuckle)
- xUnit unit tests (51 tests)
- GitHub Actions

## Setup and run instructions

**You need:** Visual Studio 2022 (with the ASP.NET workload) or the .NET 8 SDK, and SQL Server (Express, LocalDB or full).

1. Clone the repository:
   ```
   git clone https://github.com/Sinegugu03/RaceDay.git
   ```
2. Open `RaceDay.sln` in Visual Studio.
3. Open `RaceDay.Api/appsettings.json` and change `Server=` in the connection string so it matches your SQL Server instance (for example `localhost`, `.\SQLEXPRESS` or `(localdb)\MSSQLLocalDB`). The database `RaceDayApiDB` is created automatically.
4. Create the database. In the folder that contains `RaceDay.sln`, run:
   ```
   dotnet ef database update --project RaceDay.Api
   ```
   (or run `Update-Database` in the Visual Studio Package Manager Console). The API also applies the migrations when it starts.
5. Run the API with **F5** in Visual Studio, or:
   ```
   dotnet run --project RaceDay.Api --launch-profile http
   ```
6. Open Swagger at **http://localhost:5080/swagger** (or https://localhost:7080/swagger if you use the https profile).

### Demo accounts (Development only)

Password for both accounts: `Password123!`

| Email | Role |
|---|---|
| organiser@raceday.test | Organiser |
| participant@raceday.test | Participant |

### Using Swagger

1. Expand `POST /api/auth/login`, click **Try it out**, enter an email and password, and click **Execute**.
2. The browser keeps the session cookie, so every later request in Swagger is sent as that user.
3. Use `POST /api/auth/logout` to switch accounts, or open a second Incognito window to be both roles at once.

### Run the unit tests

In the folder that contains `RaceDay.sln`:
```
dotnet test
```

## How authentication works

1. `POST /api/auth/register` creates an account as an Organiser or a Participant. The password is hashed with PBKDF2 and a random salt. It is never stored or returned as plain text.
2. `POST /api/auth/login` checks the credentials and stores the user's id and role in a server-side session. The browser only holds a session cookie.
3. Every protected endpoint checks the session. If the user is not logged in the API returns **401 Unauthorized**, and if the role is wrong it returns **403 Forbidden**.
4. Organisers can only change events, categories and results that belong to them.
5. `POST /api/auth/logout` ends the session.

## Endpoints

| Area | Endpoints | Who can use them |
|---|---|---|
| Auth | `POST /api/auth/register`, `POST /api/auth/login` | Everyone |
| Auth | `POST /api/auth/logout` | Logged-in users |
| Profile | `GET` and `PUT /api/users/me` | Logged-in users |
| Profile | `GET /api/users/{id}` | Organiser |
| Events | `GET /api/events`, `GET /api/events/{id}` | Everyone |
| Events | `POST /api/events`, `PUT` and `DELETE /api/events/{id}` | Organiser (owner) |
| Categories | `GET /api/events/{eventId}/categories` | Everyone |
| Categories | `POST /api/events/{eventId}/categories`, `PUT` and `DELETE /api/categories/{id}` | Organiser (owner) |
| Enrolments | `POST /api/categories/{categoryId}/enrolments`, `GET /api/users/me/enrolments` | Participant |
| Enrolments | `GET /api/categories/{categoryId}/enrolments` | Organiser (owner) |
| Enrolments | `DELETE /api/enrolments/{id}` | Participant or Organiser (owner) |
| Results | `POST /api/enrolments/{enrolmentId}/result` | Organiser (owner) |
| Results | `GET /api/enrolments/{enrolmentId}/result` | Participant or Organiser (owner) |
| Results | `GET /api/categories/{categoryId}/results` | Everyone |

All 22 endpoints are listed in Swagger with a description, the data they expect and the responses they return.

## Differences from the Part 1 plan

- **Login** uses a session cookie instead of a JWT, because Part 2 requires session-based authentication.
- **Logout** (`POST /api/auth/logout`) was added to end a session.
- **Events** have two new fields, `EventType` (Run, Walk or Cycle) and `DistanceKm`, because the Part 2 brief requires them.
- **New enrolments** are saved with the status `Confirmed`, because there is no payment workflow in Part 2.
- **Extra 409 Conflict responses:** deleting an event or category that has enrolments, and enrolling in a closed event.
- **Past event dates** are rejected with a 400 response.
- The API database is called `RaceDayApiDB` and is created by the EF Core migrations.

## Unit tests

There are 51 xUnit tests. They start the real API in memory with an in-memory database, so no SQL Server is needed. They cover:

- registration and login (success, validation errors, duplicate email, wrong password, hashed passwords, logout)
- event and category management by Organisers
- role checks: a Participant gets 403 on Organiser endpoints, and a user who is not logged in gets 401
- enrolments (success, duplicate, full category, closed event)
- results (capture, ownership rules, public leaderboard)

## CI/CD

The GitHub Actions workflow `part2-build-test.yml` (in `.github/workflows`) restores, builds and runs all the unit tests on every push.

**Successful build screenshot:**

[Drag your green GitHub Actions screenshot here while editing this file on GitHub]

## Video walkthrough

Unlisted YouTube video showing the running API, the code structure, the endpoints, authentication and the unit tests:

PASTE YOUR YOUTUBE LINK HERE
