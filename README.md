# Ampirical Marine Space Scheduling

A full-stack marine scheduling application for reserving time slots
across a fleet of ships. The project combines a **React 19 + Vite**
frontend with a lightweight **PHP/PDO REST API** backed by **MySQL**.

The application provides two primary workflows:

-   **Marine Scheduling** --- choose a ship and date, view one-hour
    availability, enter a pilot name, and reserve an open slot.
-   **Marine Ship Listing** --- view fleet bookings grouped by ship with
    pagination and jump back to a ship's schedule.

The frontend includes a **demo/mock-data fallback**, so the interface
can still be previewed if the PHP API or database is unavailable.

------------------------------------------------------------------------

## Table of Contents

1.  [Features](#features)
2.  [Technology Stack](#technology-stack)
3.  [Project Structure](#project-structure)
4.  [Prerequisites](#prerequisites)
5.  [Quick Start](#quick-start)
6.  [Database Setup](#database-setup)
7.  [PHP API Setup](#php-api-setup)
8.  [Frontend Setup](#frontend-setup)
9.  [API Reference](#api-reference)
10. [Scheduling Rules](#scheduling-rules)
11. [Date and Time Handling](#date-and-time-handling)
12. [Demo Mode](#demo-mode)
13. [Deployment](#deployment)
14. [Troubleshooting](#troubleshooting)
15. [Current Integration Notes](#current-integration-notes)
16. [Security and Production
    Considerations](#security-and-production-considerations)
17. [Accessibility and Inclusive
    Development](#accessibility-and-inclusive-development)
18. [Contributing](#contributing)
19. [License](#license)

------------------------------------------------------------------------

## Features

### Marine Scheduling

-   Loads ships from the MySQL-backed PHP API.
-   Selects a ship and calendar date.
-   Generates one-hour booking slots from **06:00 through 22:00**.
-   Marks overlapping reservations as unavailable.
-   Requires a pilot name before a reservation can be submitted.
-   Prevents unavailable slots from being selected.
-   Displays success and error feedback after booking attempts.
-   Prevents users from selecting dates before the current local date.

### Fleet Booking View

-   Displays bookings across the fleet.
-   Sorts bookings alphabetically by ship name and then chronologically.
-   Groups displayed bookings by ship.
-   Shows booking ID, ship, pilot, start time, and end time.
-   Provides pagination with **25 bookings per page**.
-   Provides a **View Schedule** action for returning directly to a
    selected ship.

### Backend Protections

The included PHP booking endpoint:

-   Validates ship IDs.
-   Validates pilot names.
-   Validates ISO-8601 UTC timestamps.
-   Rejects bookings whose end time is not after the start time.
-   Uses a database transaction.
-   Locks the selected ship row with `FOR UPDATE`.
-   Re-checks overlapping bookings before inserting.
-   Returns HTTP `409 Conflict` when a requested slot is no longer
    available.
-   Uses PDO prepared statements.

------------------------------------------------------------------------

## Technology Stack

  Layer             Technology
  ----------------- -------------------------------------------
  Frontend          React 19
  Build tool        Vite 7
  Styling           CSS
  Backend           PHP
  Database access   PDO
  Database          MySQL
  Web server        Apache or another PHP-capable HTTP server
  Data format       JSON
  Time storage      UTC

------------------------------------------------------------------------

## Project Structure

``` text
Ampirical/
├── my-react-app/
│   ├── api/
│   │   ├── availability.php
│   │   ├── bookings.php
│   │   └── ships.php
│   ├── config/
│   │   ├── bootstrap.php
│   │   └── database.php
│   ├── public/
│   │   ├── favicon.svg
│   │   └── icons.svg
│   ├── src/
│   │   ├── main.jsx
│   │   └── styles.css
│   ├── .gitignore
│   ├── .oxlintrc.json
│   ├── index.html
│   ├── package.json
│   ├── package-lock.json
│   ├── vite.config.js
│   └── README.md
├── screenshots/
│   ├── Marine Scheudling.PNG
│   ├── Marine Viewing.PNG
│   ├── Marine Viewing 1.PNG
│   └── Marine Viewing 2.PNG
└── www/
    └── html/
        └── index.html
```

> `node_modules/` may also be present in a local/archive copy, but it
> should normally be regenerated with `npm install` rather than
> committed or deployed as source.

------------------------------------------------------------------------

## Prerequisites

For a typical Ubuntu deployment, install:

-   **Node.js** and **npm**
-   **PHP 8+**
-   PHP's **PDO MySQL** extension
-   **MySQL 8+** or a compatible MySQL server
-   **Apache 2** or another web server capable of serving PHP

Example Ubuntu packages:

``` bash
sudo apt update
sudo apt install apache2 mysql-server php libapache2-mod-php php-mysql
```

Verify the major tools:

``` bash
node --version
npm --version
php --version
mysql --version
```

------------------------------------------------------------------------

## Quick Start

The application is split into two independently served pieces:

1.  Vite serves the React development frontend, normally at
    `http://localhost:5173`.
2.  Apache/PHP serves the API at `http://localhost/api`.

The current frontend configuration is:

``` js
const API_URL = "http://localhost/api";
```

A common local arrangement is therefore:

``` text
Browser
  |
  +--> http://localhost:5173       React/Vite frontend
  |
  +--> http://localhost/api/*.php  Apache/PHP API
                                      |
                                      +--> MySQL
```

### 1. Install frontend dependencies

From the React application directory:

``` bash
cd Ampirical/my-react-app
npm install
```

### 2. Configure MySQL

Create the database, tables, and application account as described in
[Database Setup](#database-setup).

### 3. Expose the PHP files through Apache

The browser must be able to reach:

``` text
http://localhost/api/ships.php
http://localhost/api/availability.php
http://localhost/api/bookings.php
```

For example, copy or link the `api` and `config` directories into an
Apache-served application directory while preserving their relative
relationship:

``` text
/var/www/html/
├── api/
│   ├── availability.php
│   ├── bookings.php
│   └── ships.php
└── config/
    ├── bootstrap.php
    └── database.php
```

The API files expect `../config/bootstrap.php`, so keeping `api/` and
`config/` as sibling directories is important.

### 4. Start the frontend

``` bash
npm run dev
```

Open the URL printed by Vite, normally:

``` text
http://localhost:5173
```

------------------------------------------------------------------------

## Database Setup

### Expected database

By default, `config/database.php` expects:

``` text
Database: ship_booking
User:     ship_booking_app
Host:     127.0.0.1
Port:     3306
```

The password fallback in the source is `change_me`. Do **not** use that
value in production.

### Expected schema

The project expects two tables with this logical structure:

``` sql
CREATE DATABASE IF NOT EXISTS ship_booking
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE ship_booking;

CREATE TABLE ships (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB;

CREATE TABLE bookings (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    ship_id INT UNSIGNED NOT NULL,
    pilot_name VARCHAR(255) NOT NULL,
    start_time_utc DATETIME NOT NULL,
    end_time_utc DATETIME NOT NULL,
    PRIMARY KEY (id),
    CONSTRAINT fk_bookings_ship
        FOREIGN KEY (ship_id)
        REFERENCES ships(id),
    CONSTRAINT uq_booking
        UNIQUE (ship_id, pilot_name, start_time_utc, end_time_utc),
    INDEX idx_bookings_ship_time
        (ship_id, start_time_utc, end_time_utc)
) ENGINE=InnoDB;
```

> The archive does not currently contain a dedicated `.sql` schema or
> seed file. The SQL above reflects the columns and constraint names
> expected by the included application code. Review it against your
> target environment before production use.

### Optional sample ships

``` sql
INSERT INTO ships (name)
VALUES
    ('MV Atlantic Star'),
    ('MV Gulf Trader'),
    ('MV Southern Cross');
```

### Create a database user

Example:

``` sql
CREATE USER 'ship_booking_app'@'localhost'
IDENTIFIED BY 'replace_with_a_strong_password';

GRANT SELECT, INSERT, UPDATE, DELETE
ON ship_booking.*
TO 'ship_booking_app'@'localhost';

FLUSH PRIVILEGES;
```

If PHP connects through `127.0.0.1`, MySQL account host matching may
differ from `localhost` depending on your configuration. Create or grant
the account appropriate for your deployment.

------------------------------------------------------------------------

## PHP API Setup

### Database configuration

`config/database.php` reads these environment variables:

  Variable    Default
  ----------- --------------------
  `DB_HOST`   `127.0.0.1`
  `DB_PORT`   `3306`
  `DB_NAME`   `ship_booking`
  `DB_USER`   `ship_booking_app`
  `DB_PASS`   `change_me`

For production, supply credentials through the environment rather than
hard-coding secrets.

Example Apache environment configuration will vary by hosting
environment. After configuring it, restart Apache:

``` bash
sudo systemctl restart apache2
```

### Test the API

Ships:

``` bash
curl http://localhost/api/ships.php
```

Availability:

``` bash
curl "http://localhost/api/availability.php?ship_id=1&date=2026-09-27"
```

Create a booking:

``` bash
curl -X POST http://localhost/api/bookings.php \
  -H "Content-Type: application/json" \
  -d '{
    "ship_id": 1,
    "pilot_name": "Sample Pilot",
    "start_time_utc": "2026-09-27T15:00:00Z",
    "end_time_utc": "2026-09-27T16:00:00Z"
  }'
```

------------------------------------------------------------------------

## Frontend Setup

From `Ampirical/my-react-app`:

``` bash
npm install
npm run dev
```

Available npm scripts:

  Command             Purpose
  ------------------- --------------------------------------
  `npm run dev`       Start the Vite development server
  `npm run build`     Create a production frontend build
  `npm run preview`   Preview the production build locally

### API URL

The API base URL is currently hard-coded near the top of `src/main.jsx`:

``` js
const API_URL = "http://localhost/api";
```

If the backend is hosted elsewhere, update this value.

For a more portable deployment, consider replacing it with a Vite
environment variable such as `VITE_API_URL`.

------------------------------------------------------------------------

## API Reference

### `GET /api/ships.php`

Returns ships ordered alphabetically.

Example response:

``` json
[
  {
    "id": 1,
    "name": "MV Atlantic Star"
  }
]
```

### `GET /api/availability.php`

Returns bookings overlapping a selected UTC calendar day for one ship.

Query parameters:

  Parameter   Required   Description
  ----------- ---------- -----------------------------
  `ship_id`   Yes        Positive ship ID
  `date`      Yes        Date in `YYYY-MM-DD` format

Example:

``` text
GET /api/availability.php?ship_id=1&date=2026-09-27
```

Example response:

``` json
{
  "ship": {
    "id": 1,
    "name": "MV Atlantic Star"
  },
  "date": "2026-09-27",
  "timezone": "UTC",
  "bookings": [
    {
      "id": 12,
      "ship_id": 1,
      "pilot_name": "Sample Pilot",
      "start_time_utc": "2026-09-27 15:00:00",
      "end_time_utc": "2026-09-27 16:00:00"
    }
  ]
}
```

### `POST /api/bookings.php`

Creates a booking.

Example request:

``` json
{
  "ship_id": 1,
  "pilot_name": "Sample Pilot",
  "start_time_utc": "2026-09-27T15:00:00.000Z",
  "end_time_utc": "2026-09-27T16:00:00.000Z"
}
```

The frontend also sends:

``` json
{
  "run_proc": "getShipBooking"
}
```

The included PHP create endpoint currently ignores that extra field.

Successful creation returns HTTP `201`.

Example response:

``` json
{
  "message": "Booking created successfully.",
  "booking": {
    "id": 42,
    "ship_id": 1,
    "ship_name": "MV Atlantic Star",
    "pilot_name": "Sample Pilot",
    "start_time_utc": "2026-09-27 15:00:00",
    "end_time_utc": "2026-09-27 16:00:00"
  }
}
```

A scheduling conflict returns HTTP `409` with `SLOT_UNAVAILABLE`.

------------------------------------------------------------------------

## Scheduling Rules

The current slot configuration is defined in `src/main.jsx`:

``` js
const SLOT_MINUTES = 60;
const OPEN_HOUR = 6;
const CLOSE_HOUR = 22;
```

This produces one-hour slots beginning at 6:00 AM, with the final slot
ending at 10:00 PM.

A slot is unavailable when it overlaps an existing booking:

``` text
slot.start < booking.end
AND
slot.end > booking.start
```

The frontend performs this calculation for display purposes. The backend
independently checks for overlap inside a database transaction before
inserting the booking.

That backend check is essential because frontend-only validation cannot
prevent two users from attempting to reserve the same slot concurrently.

------------------------------------------------------------------------

## Date and Time Handling

The application is designed around **UTC storage**.

The frontend:

-   Builds slots using the browser's local date/time.
-   Converts selected slot times to UTC using
    `Date.prototype.toISOString()`.
-   Treats MySQL `DATETIME` values returned without a timezone suffix as
    UTC.
-   Displays booking timestamps using the browser's local timezone.

The PHP API:

-   Validates incoming UTC ISO-8601 timestamps.
-   Converts them to MySQL `Y-m-d H:i:s` values in UTC.
-   Returns availability for bookings overlapping the requested UTC
    calendar date.

For multi-timezone production deployments, verify that the intended
meaning of the selected calendar date matches the operating location's
timezone.

------------------------------------------------------------------------

## Demo Mode

If the frontend cannot load the API, it automatically switches to mock
data.

The header displays:

``` text
Demo Data
```

Mock ships include:

-   MV Atlantic Star
-   MV Gulf Trader
-   MV Southern Cross

Bookings created while demo mode is active exist only in React state and
are lost when the page reloads.

Demo mode is useful for UI development, but it can also hide backend
configuration failures. Check the browser developer console and API URLs
when you expect live database data.

------------------------------------------------------------------------

## Deployment

### Build the frontend

``` bash
cd Ampirical/my-react-app
npm install
npm run build
```

Vite writes the production frontend to:

``` text
dist/
```

### Apache example

A simple deployment can serve the compiled frontend from Apache while
keeping PHP endpoints under `/api`.

One possible layout:

``` text
/var/www/html/
├── index.html
├── assets/
├── api/
│   ├── availability.php
│   ├── bookings.php
│   └── ships.php
└── config/
    ├── bootstrap.php
    └── database.php
```

Copy the contents of `dist/` into the web root, then add the PHP
directories.

Because the frontend currently uses:

``` js
const API_URL = "http://localhost/api";
```

change the API URL before building when the deployed hostname is not
`localhost`. A relative `/api` URL is often simpler when the frontend
and API share the same origin.

------------------------------------------------------------------------

## Troubleshooting

### `http://localhost:5173/api/bookings` returns 404

The current Vite configuration does **not** define an `/api` proxy. The
application source calls:

``` text
http://localhost/api/*.php
```

not:

``` text
http://localhost:5173/api/*
```

Make sure Apache/PHP is running and serving the PHP files at
`http://localhost/api`.

Alternatively, add a Vite development proxy and change the frontend API
configuration accordingly.

### The UI says `Demo Data`

At least one API request failed.

Check:

``` bash
curl http://localhost/api/ships.php
```

Also inspect the browser developer console and Network tab.

Common causes include:

-   Apache is not running.
-   PHP files are not under the expected web root.
-   The API URL is incorrect.
-   MySQL is unavailable.
-   Database credentials are incorrect.
-   `php-mysql` is not installed.
-   The database or expected tables do not exist.

### Check Apache

``` bash
sudo systemctl status apache2
```

Restart it with:

``` bash
sudo systemctl restart apache2
```

### Check MySQL

``` bash
sudo systemctl status mysql
```

### Verify PDO MySQL

``` bash
php -m | grep -i pdo
php -m | grep -i mysql
```

### PHP returns a database error

Review the Apache/PHP logs. On many Ubuntu installations:

``` bash
sudo tail -f /var/log/apache2/error.log
```

Do not expose raw database exception messages to end users in
production.

### CORS problems

`config/bootstrap.php` currently sends:

``` text
Access-Control-Allow-Origin: *
```

This permits cross-origin frontend requests during development. In
production, restrict CORS to trusted origins when cross-origin access is
required.

------------------------------------------------------------------------

## Current Integration Notes

### Fleet Listing API mismatch

`src/main.jsx` loads the fleet listing by sending:

``` json
{
  "run_proc": "getAllBookings"
}
```

to:

``` text
POST /api/bookings.php
```

However, the included `api/bookings.php` currently handles every POST as
a **create booking** request and requires `ship_id`, `pilot_name`,
`start_time_utc`, and `end_time_utc`.

As a result, the live **Marine Ship Listing** request will fail with the
current PHP endpoint and the UI will fall back to mock bookings.

Before production use, implement a list-all-bookings endpoint or add
explicit routing for `run_proc: "getAllBookings"`.

A typical list query would join ships and bookings:

``` sql
SELECT
    b.id,
    b.ship_id,
    s.name AS ship_name,
    b.pilot_name,
    b.start_time_utc,
    b.end_time_utc
FROM bookings b
JOIN ships s ON s.id = b.ship_id
ORDER BY s.name ASC, b.start_time_utc ASC;
```

### No schema migration file

The archive does not currently include a `.sql` migration/schema file.
Add one to source control so new environments can reproduce the database
consistently.

### No automated test suite

The current `package.json` does not define frontend or backend test
scripts. Production development would benefit from tests covering slot
overlap, date conversion, validation, API errors, and concurrent booking
attempts.

------------------------------------------------------------------------

## Security and Production Considerations

Before exposing the application publicly, consider the following:

-   Replace the fallback database password and keep secrets outside
    source control.
-   Restrict CORS instead of using `Access-Control-Allow-Origin: *`.
-   Serve the application over HTTPS.
-   Add authentication and authorization if booking data is not public.
-   Apply least-privilege MySQL permissions.
-   Add rate limiting where appropriate.
-   Validate and log API failures without exposing sensitive exception
    details.
-   Consider CSRF protections depending on the authentication design.
-   Define the authoritative operational timezone.
-   Add database migrations and backups.
-   Configure production logging and monitoring.
-   Review whether pilot names are personal data under your
    organization's privacy requirements.
-   Do not commit `node_modules/`, credentials, local environment files,
    or database dumps containing sensitive information.

------------------------------------------------------------------------

## Accessibility and Inclusive Development

Contributions should aim to keep the application usable across devices,
input methods, and user needs.

Current positive foundations include native form controls, associated
`label` elements, real buttons for interactive actions, disabled states,
and responsive layouts.

When extending the UI:

-   Preserve keyboard navigation.
-   Maintain visible focus indicators.
-   Do not communicate status by color alone.
-   Use clear, descriptive labels and error messages.
-   Maintain sufficient text/background contrast.
-   Add appropriate table captions or accessible context when needed.
-   Test layouts at narrow widths and with browser zoom.
-   Test with screen readers for major workflows.
-   Use respectful, role-appropriate language and avoid assumptions
    about users.

------------------------------------------------------------------------

## Contributing

Contributions are welcome when they improve reliability,
maintainability, accessibility, security, documentation, or user
experience.

A suggested workflow:

``` bash
git checkout -b feature/short-description
npm install
npm run dev
```

Before submitting a change:

1.  Confirm the frontend builds successfully with `npm run build`.
2.  Test the scheduling workflow with the live PHP/MySQL backend.
3.  Test conflict handling by attempting overlapping reservations.
4.  Verify the fleet listing if its backend endpoint has been
    implemented.
5.  Check responsive behavior on desktop and mobile widths.
6.  Avoid committing secrets or generated dependency directories.
7.  Update this README when setup, API contracts, or behavior changes.

Use clear commit messages that describe the purpose of the change.

------------------------------------------------------------------------

## License

No license file was found in the supplied project archive.

Until a license is explicitly added, do not assume the project grants
public permission to copy, modify, or redistribute the source code.
Project owners should add an appropriate `LICENSE` file if the
repository is intended for distribution or open-source collaboration.
