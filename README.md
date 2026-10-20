# LinkPulse

**A URL shortener with per-user link management, click and visitor tracking, and an admin dashboard.**

Built for the *LinkPulse: URL Shortener & Analytics Platform* hackathon problem statement.
The brief asks for a working, deployed core rather than a long list of half-built features, so this project focuses on a small set of features that work end to end.

- **Live demo:** `<add your Render URL here>`
- **Admin login:** username `admin`. The password is set with the `ADMIN_PASSWORD` environment variable and is shared with judges privately.
- **Team:** `<add team name and members>`

---

## What it does

LinkPulse has three kinds of people, as in the problem statement:

| Role | What they can do |
|---|---|
| **Visitor** (not logged in) | Open any short link. No management access. |
| **User** | Sign up, shorten links, and see clicks and unique visitors for **their own links only**. |
| **Admin** | See all users, how many links each created, which devices they use, total clicks. Block, unblock or delete a user. |

### User experience
- Paste a long link (Ctrl + V or the Paste button) and press Shorten to get a short link.
- A list of the user's own links shows the **clicks** and **visitors** for each one. It refreshes automatically every 5 seconds.
- A user cannot see anyone else's links or data.

### Admin experience
- Totals at the top: number of users, links created, and total clicks.
- A table with one row per user: username, devices used (for example "Mobile (Android, Chrome)"), joined date, links created, clicks, and status.
- **Block** logs the user out immediately and stops them from logging in until unblocked.
- **Delete** removes the account and all of its links. Both actions ask for confirmation in an in-app dialog.
- The admin cannot block or delete the admin account.

---

## How it maps to the problem statement

| Requirement from the brief | Status in this version |
|---|---|
| Short link creation with unique auto-generated codes | **Done** (6-character random codes, collision-checked) |
| Fast, reliable redirection | **Done** (302 redirect, counters kept in memory, disk written in the background) |
| Click tracking | **Done, basic.** Total clicks and unique visitors per link |
| Role-based access (Visitor, User, Admin) | **Done** (enforced on the server for every protected route) |
| URL validation before publishing | **Done** (valid http/https URL only, no links to the app's own domain) |
| Admin can disable abusive users | **Done** (block, unblock, delete accounts) |
| Dashboard uses live data, nothing hardcoded | **Done** (no sample data is seeded; both dashboards read real data) |
| Hosted link (minimum requirement) | **Done** (deployed on Render) |
| Custom aliases, expiry dates, click limits, password-protected links | Roadmap |
| Per-click details: time, referrer, device, browser, location | Roadmap |
| Bot filtering | Roadmap |
| Campaigns, tags, workspaces, folders | Roadmap |
| QR codes, CSV bulk import, link-in-bio pages | Roadmap |
| Rate limiting on link creation | Roadmap |
| Domain blocklist, device targeting, public API, custom domains | Roadmap |

---

## Tech stack

| Layer | Choice |
|---|---|
| Frontend | HTML, CSS and vanilla JavaScript (no framework, no build step) |
| Backend | Node.js with Express |
| Storage | JSON file, loaded into memory and saved to disk every 2 seconds |
| Auth | Password hashing with scrypt, random session tokens, role checks on each request |
| Hosting | Render (Web Service) |

---

## Project structure

```
linkpulse/
├── server.js            Backend: accounts, links, admin actions, redirect and tracking
├── package.json         Dependencies (express) and the start command
├── README.md            This file
├── data/
│   └── linktrack.json   Storage, created automatically on first run (not committed)
└── public/              Frontend, served by the same server
    ├── index.html       Page shell
    ├── css/style.css    Design
    └── js/app.js        Login screen, user dashboard, admin dashboard, confirm dialog
```

---

## Data model

```
users[username]  = { role, created, devices[], blocked, salt, hash }
sessions[token]  = username
links[code]      = { code, url, owner, created, clicks, vids[] }
```

- One user owns many links (`links[code].owner` points to a username).
- `vids` holds anonymous visitor IDs. A visitor ID is the first 12 characters of a SHA-256 hash of the visitor's IP address and browser string. Unique visitors are the number of distinct IDs.
- Deleting a user also deletes their links and sessions.

## How a click is handled

```
Visitor opens  https://<host>/aB3xYz
        |
        v
Server finds the link by code  --- not found --->  "Link not found" (404)
        |
        v
Visitor ID = hash(IP + browser)
clicks + 1, add the visitor ID if it is new
        |
        v
302 redirect to the original URL      (disk write happens later, in the background)
        |
        v
The owner's dashboard shows the new numbers on its next refresh
```

---

## API

| Method and path | Who can call it | Purpose |
|---|---|---|
| `POST /api/register` | Anyone | Create a user account and log in |
| `POST /api/login` | Anyone | Log in (blocked users are refused) |
| `POST /api/logout` | Logged in | End the session on the server |
| `GET /api/me` | Logged in | Current username and role |
| `POST /api/links` | User | Create a short link |
| `GET /api/links` | User | List own links with clicks and visitors |
| `GET /api/admin/summary` | Admin | Totals and the list of users |
| `POST /api/admin/users/:name/block` | Admin | Block or unblock a user (`{ "blocked": true }`) |
| `DELETE /api/admin/users/:name` | Admin | Delete a user and their links |
| `GET /:code` | Anyone | Redirect and record the click |

Protected routes expect the header `Authorization: Bearer <token>`.

---

## Security notes

- Passwords are hashed with scrypt and a per-user salt, and compared in constant time. They are never stored or logged in plain text.
- Every protected route checks the session and the role on the server. Hiding a button in the page is not what protects admin features.
- Usernames are lowercased, so `Admin` and `admin` cannot be two different accounts.
- Lookups use own-property checks, so names like `constructor` cannot break the server.
- All user-supplied text is escaped before it is shown in the page.
- Only `http` and `https` destinations are accepted.
- The admin password comes from the `ADMIN_PASSWORD` environment variable, not from the repository.

---

## Run locally

```bash
npm install
npm start
```

Open `http://localhost:3000`. If the port is busy, set `PORT`, for example `PORT=3001 npm start` (on Windows PowerShell: `$env:PORT=3001; npm start`).

The first start creates the `admin` account. Set `ADMIN_PASSWORD` before that first start to choose its password. If it is not set, the default is `admin123`, which is only suitable for local testing.

## Deploy on Render

1. Push this project to a GitHub repository (without `node_modules` and `data`).
2. In Render choose **New + > Web Service** and connect the repository.
3. Use **Build Command** `npm install` and **Start Command** `npm start`.
4. In the **Environment** tab add `ADMIN_PASSWORD` with a strong password.
5. Deploy. Render gives a public `https://...onrender.com` URL that works from any device and network.

---

## Suggested demo (5 minutes)

**Scenario 1: user creates a link and sees analytics**
1. Create an account, paste a long link, press Shorten.
2. Open the short link on a phone using mobile data.
3. Back on the dashboard, clicks and visitors increase.

**Scenario 2: admin manages users**
1. Log in as `admin`. The dashboard shows the new user, their device, and their link count.
2. Block the user. Their session ends, and logging in shows the blocked message.
3. Unblock, then delete the user. Their short link now returns "Link not found".

---

## Known limitations

- **Storage is a JSON file.** On Render's free plan the filesystem is reset when the service restarts or redeploys, so users and links are lost. A real deployment needs a persistent database.
- **The free plan sleeps** after a period of inactivity, so the first request afterwards is slow.
- **Tracking is basic.** Only totals are stored, not a log of each click. Because of that, time, referrer, device, browser and location of each click are not available yet.
- **The device shown for a user** is the device they signed up or logged in from (the last 5 distinct ones), not the devices of people who open their links.
- **Link previews** from chat apps and crawlers are counted as clicks, as there is no bot filtering yet.
- **Unique visitors are an approximation.** The same person on a different network or browser counts as a new visitor.
- A blocked user's short links keep working. Only their account access is blocked.
- Sessions do not expire on their own, and there is no password reset.

## Roadmap

1. Persistent database (PostgreSQL or MongoDB) instead of the JSON file.
2. A click-event log (time, referrer, device, browser, location) with bot filtering, and charts built from it.
3. Custom aliases, expiry dates, click limits and password-protected links, enforced by the redirect engine.
4. Campaigns and tags, then QR codes with scan tracking and CSV bulk import.
5. Rate limiting on link creation and a domain blocklist managed by the admin.
6. Device targeting, a public API, and custom domains.
