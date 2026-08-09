# Running SwasthBharat locally, and pushing changes

Practical setup and git guide. For what the project *is* and why, see [`README.md`](./README.md).

## The one thing that trips everybody up

This repository has **two different working directories**, and using the wrong one is the
most common failure:

```
swatchbharat3/                                          <-- git root. Run all GIT commands here.
├── README.md
├── DEVELOPING.md
└── SwasthBharat-b0a58f121e010d5422105d2f71cb400eea165949/   <-- app root. Run all NPM commands here.
    ├── package.json          (defines setup, dev:api, dev:web, check:*)
    ├── backend/              Express API, Socket.io, MongoDB models
    ├── frontend/             TypeScript/React PWA
    ├── shared/               risk engine + chatbot rules (used by both)
    ├── ml/                   model training and the JS export it produces
    └── api/index.js          Vercel serverless entry point
```

Running `npm run setup` from the git root fails with `npm error Missing script: "setup"`,
because the root `package.json` is not the project's. `cd` into the long-named folder first.

For brevity, the rest of this guide calls that folder **the app root**.

## Prerequisites

- **Node.js 20 or newer** (`node --version`). Enforced via `engines` in every `package.json`.
- **git**.
- Python is *not* required. The trained model ships as plain JavaScript in `shared/risk/`.
- No MongoDB install needed for local dev — see the database note below.

## First-time setup

From the **app root**:

### 1. Install dependencies

```powershell
npm run setup
```

This runs `npm install` separately for `backend/` and `frontend/`. They are deliberately not
npm workspaces — the root `package.json` documents the reasoning under `comments.why-no-workspaces`.
Expect roughly 340 backend and 450 frontend packages.

### 2. Create your env file

```powershell
Copy-Item backend/.env.example backend/.env     # PowerShell
```

```bash
cp backend/.env.example backend/.env            # macOS / Linux / git-bash
```

Watch the spacing in PowerShell. `Copy-Item backend/.env.example backend/ .env` (stray space
before `.env`) fails with `A positional parameter cannot be found that accepts argument '.env'`.

### 3. Edit `backend/.env` — two required changes

A raw copy of `.env.example` **will not boot**. It ships a placeholder Atlas URI with
`USE_IN_MEMORY_DB=false`, so startup dies with
`querySrv ENOTFOUND _mongodb._tcp.cluster0.xxxxx.mongodb.net`. Change both of these:

```ini
USE_IN_MEMORY_DB=true
JWT_SECRET=<paste a long random string>
```

`USE_IN_MEMORY_DB=true` makes `MONGO_URI` ignored entirely, so you can leave the placeholder
alone. First boot downloads a `mongod` binary (~130 MB) into `.cache/`, so do this **before**
a demo, not during one. Data is discarded on every shutdown, which is fine because the API
re-seeds an empty database automatically at startup.

Generate the secret with:

```powershell
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

Optional: set `SETUP_TOKEN` to another random string. It is only needed to create doctor or
district-officer accounts through `/api/auth/register`. The demo accounts below don't need it.

`frontend/.env.local` is entirely optional — copy `frontend/.env.example` only if you want to
change defaults. Requests stay same-origin and Vite proxies `/api` to port 4000.

## Running it

Two terminals, both from the app root. There is intentionally no combined `dev` script, so
that a crash in one is not hidden by the other.

```powershell
npm run dev:api      # terminal 1 -> http://localhost:4000
npm run dev:web      # terminal 2 -> http://localhost:5173
```

The API is ready when it prints `SwasthBharat API listening on http://localhost:4000`. Confirm
with:

```powershell
(Invoke-WebRequest http://localhost:4000/api/health -UseBasicParsing).Content
```

A healthy response reports `"status":"ok"` and `"database":{"state":"connected","inMemory":true}`.

Then open <http://localhost:5173>.

### Demo logins

Seeded automatically on first API startup. Password is `demo1234` for all of them.

| Role | Phone | Name | PHC |
| --- | --- | --- | --- |
| ASHA worker (Bengali) | `9800000001` | Sunita Das | Haringhata |
| ASHA worker (Hindi) | `9800000002` | Rekha Kumari | Chakdaha |
| ASHA worker (Bengali) | `9800000003` | Aparna Mondal | Krishnanagar |
| PHC doctor | `9800000010` | Dr. Arun Ghosh | Haringhata |
| PHC doctor | `9800000011` | Dr. Ravi Sharma | Chakdaha |
| District health officer | `9800000020` | Dr. Meera Nair | Nadia district |

To see the live high-risk alert, log in as `9800000001` and `9800000010` in two windows — same
PHC — then submit a high-risk screening as the worker.

### Demoing offline mode and PWA install

Use the production build, not the dev server:

```powershell
npm run demo         # builds, then serves on http://localhost:4173
```

Offline reload and "install app" rely on a precached service worker. Vite's dev server serves
modules on demand, so neither works on port 5173. Both ports are already in the default
`CORS_ORIGINS`.

## Verification

```powershell
npm run check:api        # full demo flow, 79 assertions (API must be running)
npm run check:security   # cross-PHC access denial and rate limiting
npm run check:web        # TypeScript typecheck
npm run check:i18n       # Bengali/Hindi translations against English
```

Run `check:security` **last**. It deliberately exhausts the login rate limiter (30 requests per
10 minutes per IP), so any login right after it gets `TOO_MANY_ATTEMPTS` until the window rolls
over. Restart the API to reset it immediately.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `npm error Missing script: "setup"` | You're in the git root. `cd` into the app root. |
| `A positional parameter cannot be found that accepts argument '.env'` | Stray space in `Copy-Item`. Target is `backend/.env`, one argument. |
| `querySrv ENOTFOUND _mongodb._tcp.cluster0.xxxxx.mongodb.net` | `.env` still has the placeholder Atlas URI. Set `USE_IN_MEMORY_DB=true`. |
| First `dev:api` sits silently for a minute | Downloading the `mongod` binary. Normal, once only. |
| Login fails with "Could not log in" but curl works | Your browser origin isn't in `CORS_ORIGINS`. Port 4173 must be listed even though Vite proxies `/api`. |
| `EADDRINUSE` on 4000 | Another API instance is running. `Get-Process node \| Stop-Process` or change `PORT`. |
| Warning that `JWT_SECRET` is the placeholder | Harmless locally, fatal in production — the server refuses to start. Set a real one. |

## Pushing changes

**Run every git command from the git root** (`swatchbharat3`), not the app root.

### Check where you're pushing

```powershell
git remote -v
git branch -vv
```

This checkout currently has two remotes: `origin` pointing at `AbhinavKM510/swatchbharat3`, and
`odinkiller` pointing at `odinkiller/SwasthBharat`. Confirm which one you mean before pushing.

### Always fetch before you push

The remote moves independently, so a push can be rejected as non-fast-forward:

```powershell
git fetch odinkiller
git log --oneline HEAD..odinkiller/main     # commits you don't have yet
git log --oneline odinkiller/main..HEAD     # commits you'd be pushing
```

If the first command lists anything, integrate before pushing:

```powershell
git merge --ff-only odinkiller/main    # clean when you have no local commits
git rebase odinkiller/main             # when you do have local commits
```

If you have uncommitted work in the way, park it first — `git stash push -m "wip"`, integrate,
then `git stash pop`.

**Never resolve a rejected push with `--force`.** A force push here would delete whatever the
remote gained since you last fetched. Fetch and rebase instead.

### Commit and push

```powershell
git add <specific files>
git commit -m "Describe the change"
git push odinkiller main
```

Stage files by name rather than `git add .`, which is how build artifacts and stray local config
end up in history.

### Two things not to commit

**Secrets.** `.env`, `.env.local` and `.firebaserc` are gitignored (see the app root
`.gitignore`). Keep it that way — `.env` holds your `JWT_SECRET` and `SETUP_TOKEN`. Verify with:

```powershell
git check-ignore -v SwasthBharat-b0a58f121e010d5422105d2f71cb400eea165949/backend/.env
```

**The npm `file:..` artifact.** `npm run setup` sometimes injects `"swasthbharat": "file:.."`
into `backend/package.json` and `frontend/package.json`. That is a local install side effect,
not a project dependency. If `git status` shows those two files modified right after installing
and you changed nothing yourself, discard them:

```powershell
git checkout -- SwasthBharat-b0a58f121e010d5422105d2f71cb400eea165949/backend/package.json SwasthBharat-b0a58f121e010d5422105d2f71cb400eea165949/frontend/package.json
```

## Deploying

Vercel config is already committed — no second host needed:

- `vercel.json` at the app root rewrites `/api/(.*)` to `api/index.js`, which wraps the same
  `createApp()` the local server uses.
- `frontend/vercel.json` handles SPA routing for the PWA.

Deploy as two Vercel projects from the same repo: one with Root Directory set to the app root
(the API), one set to `frontend/` (the PWA).

Two things differ from local:

- **`USE_IN_MEMORY_DB` cannot be used.** A serverless function is frozen between requests, so
  set a real `MONGO_URI` (Atlas) plus `JWT_SECRET` and `SETUP_TOKEN` as Vercel environment
  variables. Add Vercel's egress to the Atlas IP allow-list. A misconfiguration here returns
  `503 DATABASE_UNAVAILABLE` with the specific reason in the response.
- **Socket.io is not initialised.** A function can't hold a WebSocket open. The emit helpers in
  `backend/src/realtime/io.js` no-op safely, and the doctor dashboard polls instead. Live push
  still works when you run the API locally.
