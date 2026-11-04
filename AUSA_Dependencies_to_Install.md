# AUSA E-Voting System: Everything You Need to Install

Version 1, November 2, 2026. Matches the Backend Dependencies and API Contract v4.1 and Blueprint v4.

This file answers one question: **what must be on my laptop, my server and my accounts before I can work?** Do the parts in order. Every command is for **Windows PowerShell** unless a section says it is for the server.

> Rule of thumb: install, then run the check command next to it. If the check fails, fix it before moving on. A half-working install costs days later.

---

## Part 1. Accounts to open (no installing, but work is blocked without them)

| Account | Who | Why | Needed by |
|---|---|---|---|
| GitHub | Both leads | Two repositories: `backend` and `frontend`. Protected `main`, pull requests reviewed by the other lead. | Week 0 |
| Personal Gmail with 2-step verification and an **app password** | Backend lead | Sends test OTP emails while building (about 500 a day). Use your own address in `.env`. The sample `ausa.test@gmail.com` in the files is only a placeholder. | Start of week 2 |
| Cloudflare (free plan) | Backend lead | Protects the public address (Option A, or Option B if the school's DNS is on Cloudflare). | Week 2 |
| Server provider (DigitalOcean or Lineserve) | Backend lead | One small staging server in week 2, a bigger production server in week 7. | Staging: week 3 latest |
| Domain (Option A) or a request to ICT for a subdomain (Option B) | Backend lead | The one public address. Staging needs HTTPS, and HTTPS needs an address. | Start of week 2 |
| School Google Workspace account from ICT | Backend lead, through the school | Sends the real OTP emails. Ask for a **dedicated** account (for example `no-reply-evote@` on the school domain), not an alias, plus an app password or the SMTP relay. | Ask in week 0, decide by end of week 5, hard limit week 7 |
| Sentry (free tier) | Backend lead | Error alerts on the live server. | Week 7 |
| Figma access | Frontend lead | The v4 screens. | Week 0 |
| Authenticator app on your phone (Google Authenticator, Microsoft Authenticator, or similar) | Admin and officers | Staff login needs a code from it. | Week 2 |

---

## Part 2. Tools to install on a Windows laptop

### 2.1 Who installs what

| Tool | Backend lead | Frontend lead | Notes |
|---|---|---|---|
| Git | Yes | Yes | |
| Python 3.12 | Yes | No | Django 5.2 supports 3.10 to 3.13. Use 3.12 everywhere so laptop and server match. |
| Docker Desktop (WSL2) | Yes | Yes | Runs the database. Needs virtualization turned on in BIOS on some laptops. |
| Node.js LTS (22 or newer) | No | Yes | For React. |
| VS Code | Yes | Yes | Any editor works. |
| An API tester (Postman or Bruno) | Yes | Optional | To try endpoints by hand. |

### 2.2 Install with winget (PowerShell, run as normal user, accept prompts)

```powershell
winget install --id Git.Git -e
winget install --id Python.Python.3.12 -e
winget install --id Docker.DockerDesktop -e
winget install --id OpenJS.NodeJS.LTS -e
winget install --id Microsoft.VisualStudioCode -e
```

Close PowerShell and open it again after installing, so the new tools are found.

If Docker Desktop asks for WSL2, run this once in an **administrator** PowerShell and restart the laptop:

```powershell
wsl --install
```

### 2.3 Check each install worked

```powershell
git --version
py -3.12 --version
docker --version
docker compose version
node -v
npm -v
```

Every line must print a version. For Docker, also open **Docker Desktop** and wait until it says it is running before you use `docker` commands.

### 2.4 Do NOT install or use these

| Do not use | Why | Use instead |
|---|---|---|
| XAMPP's MariaDB | Many XAMPP builds ship MariaDB 10.4. Django 5.2 refuses it. | MariaDB 10.11 in Docker |
| Django 6.x | Some add-ons only list support up to 6.0. Do not jump mid-project. | Django 5.2 LTS |
| django-cors-headers | The frontend and the API share one address, so CORS is not needed. | Nothing. Do not install it. |
| Nginx and certbot | Replaced in v4.1. | Caddy in Docker |
| A different package version "because it is newer" | The versions below were checked together. | Ask first, then change |

---

## Part 3. Backend (Python) packages

The backend lead does this inside the backend project folder.

### 3.1 Create the environment

```powershell
py -3.12 -m venv venv
venv\Scripts\activate
python -m pip install --upgrade pip
```

You should see `(venv)` at the start of the prompt. Do this every time you open a new PowerShell window: `venv\Scripts\activate`.

### 3.2 `requirements.txt` (what runs in production)

```text
Django==5.2.18
djangorestframework==3.18.3
mysqlclient==2.3.0
django-environ==0.14.0
django-axes==8.3.1
drf-spectacular==0.30.0
Pillow==12.3.0
nh3==0.3.7
argon2-cffi==25.1.0
django-otp==1.7.4
gunicorn==26.2.0
whitenoise==6.12.0
```

### 3.3 Only if you need them

```text
# Only if the live email sender becomes Amazon SES (not needed for Gmail or Google Workspace)
django-anymail[amazon-ses]==15.2

# Only if you add the email queue
celery==5.6.3
redis==8.1.0
```

### 3.4 `requirements-dev.txt` (tests and checks, laptop only)

```text
-r requirements.txt
pytest==9.1.1
pytest-django==4.14.0
factory-boy==3.3.3
pip-audit==2.10.1
```

### 3.5 Install and check

```powershell
pip install -r requirements-dev.txt
pip check
pip list
```

`pip check` must say **No broken requirements found**.

### 3.6 If `mysqlclient` will not install on Windows

Use PyMySQL on the **laptop only**. The server keeps `mysqlclient`.

```powershell
pip install PyMySQL==1.2.3
```

Then add these two lines at the very top of `config/__init__.py`:

```python
import pymysql
pymysql.install_as_MySQLdb()
```

### 3.7 What each package is for (in plain words)

| Package | What it does for us |
|---|---|
| Django | The main framework: database models, sessions, login, password hashing, the admin site. |
| djangorestframework | Turns Django into a JSON API with permissions and rate limits. |
| mysqlclient | Lets Django talk to MariaDB. |
| django-environ | Reads secrets from the `.env` file so none sit in the code. |
| django-axes | Locks repeated wrong logins (on matric number plus IP, so nobody can lock out a real student). |
| drf-spectacular | Builds the live API docs page at `/api/docs/`, so the contract can be checked against the real code. |
| Pillow | Checks that uploaded candidate photos are really images. |
| nh3 | Cleans blog text so dangerous code cannot be saved. |
| argon2-cffi | Stronger password hashing. |
| django-otp | Authenticator-app codes for admin and officer logins. Free, no SMS needed. |
| gunicorn | Runs Django on the server. |
| whitenoise | Serves Django's own static files. |
| pytest, pytest-django, factory-boy | Tests and fake data. |
| pip-audit | Checks packages for known security problems before each release. |

---

## Part 4. Database (MariaDB 10.11 in Docker)

This replaces XAMPP's database. Docker Desktop must be running.

### 4.1 Start the laptop database

The file `deploy/compose.dev.yaml` is in the contract file (section A7). Save it in the backend project, then:

```powershell
docker compose -f deploy/compose.dev.yaml up -d
docker ps
```

`docker ps` should show a `mariadb:10.11` container running. The database listens on `127.0.0.1` only, so nothing outside your laptop can reach it.

### 4.2 Point Django at it

In `.env` (never commit this file):

```dotenv
DATABASE_URL=mysql://evote_user:devpassword@127.0.0.1:3306/evote
```

Then:

```powershell
python manage.py migrate
```

### 4.3 Stop or reset it

```powershell
docker compose -f deploy/compose.dev.yaml down
```

Add `-v` at the end only if you want to **delete all data** in the dev database.

---

## Part 5. Frontend (React) packages

The frontend lead does this. These exact versions were installed together on November 2, 2026 with no conflicts and `npm audit` reported 0 problems. The Vite scaffold step and a full app build were **not** tested, so run them yourself and tell the backend lead if anything breaks.

### 5.1 Create the project

```powershell
npm create vite@latest frontend -- --template react-ts
cd frontend
```

### 5.2 Install the exact versions

Run each command on one line.

```powershell
npm install --save-exact react@19.3.1 react-dom@19.3.1 react-router-dom@7.18.5 axios@1.21.0 @tanstack/react-query@5.105.1 react-hook-form@7.91.0 zod@4.6.7 @hookform/resolvers@7.0.1 dompurify@3.4.17
```

```powershell
npm install --save-dev --save-exact vite@8.3.5 typescript@7.0.2 vitest@5.0.5 msw@3.0.3 @testing-library/react@16.3.3 @testing-library/user-event@14.6.8 @testing-library/jest-dom@7.0.1 jsdom@30.1.3 @vitejs/plugin-react@6.2.1 @types/react@19.3.0 @types/react-dom@19.3.0
```

Optional styling (Tailwind):

```powershell
npm install --save-dev --save-exact tailwindcss@4.3.3 @tailwindcss/vite@4.3.3
```

Set up the mock server that lets you build before the backend is ready:

```powershell
npx msw init public/ --save
```

### 5.3 Check

```powershell
npm ls --depth=0
npm audit
```

Commit `package.json` and `package-lock.json`. From then on, install with `npm ci` so everyone gets identical versions.

### 5.4 What each package is for

| Package | What it does for us |
|---|---|
| React with Vite and TypeScript | The app and the fast build tool. |
| react-router-dom | Pages and routes. |
| axios | API calls. Must send cookies and the `X-CSRFToken` header. |
| @tanstack/react-query | Loading and refreshing server data. Do **not** cache the token responses. |
| react-hook-form with zod and @hookform/resolvers | Forms and validation. |
| dompurify | Cleans any HTML before showing it. Never use `dangerouslySetInnerHTML` without it. |
| msw (Mock Service Worker) | Fake API answers copied from the contract, so you can build from week 1. |
| vitest with Testing Library and jsdom | Tests. |
| Tailwind CSS (optional) | Styling. |

---

## Part 6. Server software (staging and production)

The servers run Ubuntu 24.04, so these steps use Linux commands. They are written out in the contract file (sections A7 and A8). This list is only what gets installed.

| Software | How | Purpose |
|---|---|---|
| Docker Engine and the Compose plugin | Install on the server once | Runs everything below as containers. |
| `mariadb:10.11` container | From `deploy/compose.yaml` | The database. No public port, ever. |
| `web` container (Python 3.12, Gunicorn) | Built from the `Dockerfile` | Django and the API. |
| `caddy:2` container | From `deploy/compose.yaml` | Serves the React build, sends `/api/*` to Django, gets and renews HTTPS by itself. |
| Firewall | Open ports 80 and 443 only. SSH only from your own IP, key login only. | Keeps strangers out. |
| Daily database backup (cron) | Dump, then copy **off** the server | Recovery. Test one real restore. |
| GitHub Actions | In the backend repository | Tests must pass before anything deploys to staging. |
| Sentry (free tier) and an uptime check | Week 7 | Alerts both leads when something breaks. |

---

## Part 7. Settings file you will need: `.env`

Copy the full `.env.example` from section A6 of the contract file. The keys, in short:

| Key | What it is |
|---|---|
| `DJANGO_SECRET_KEY`, `OTP_HMAC_KEY`, `TOKEN_HMAC_KEY` | Three different long random strings. Never reuse one for another. |
| `DJANGO_DEBUG` | `False` on servers. |
| `SITE_ADDRESS`, `DJANGO_ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS` | The one public address. |
| `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_ROOT_PASSWORD`, `DATABASE_URL` | Database access. On laptops the host is `127.0.0.1`; on servers it is `db`. |
| `EMAIL_*`, `DEFAULT_FROM_EMAIL`, `OTP_DAILY_SEND_LIMIT`, `OTP_GLOBAL_HOURLY_LIMIT` | Email sender. Only these change when switching to the school account. |
| `ALLOW_ABSTAIN`, `HELPDESK_REISSUE_DURING_LIVE`, `SIGNUP_MISMATCH_LIMIT_PER_HOUR`, `SESSION_COOKIE_AGE` | Rules that can change without code changes. |

Make sure `.env` is listed in `.gitignore` **before** your first commit.

---

## Part 8. Quick problem solver

| Problem | Likely cause | Fix |
|---|---|---|
| `docker` command not found or cannot connect | Docker Desktop is not running | Open Docker Desktop and wait until it says running. |
| Docker will not start | Virtualization is off | Turn on virtualization in BIOS, then run `wsl --install` as administrator. |
| `mysqlclient` fails to install | Missing build tools on Windows | Use the PyMySQL fallback in 3.6 (laptop only). |
| Django says MariaDB version is too old | You are using XAMPP's MariaDB 10.4 | Stop XAMPP's MySQL and use the Docker database. |
| `py -3.12` not found | Python 3.12 not installed or terminal not restarted | Install it, close and reopen PowerShell. |
| `venv\Scripts\activate` is blocked | PowerShell script policy | Run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once, then try again. |
| Port 3306 already in use | XAMPP MySQL is running | Stop it in the XAMPP control panel. |

---

## Part 9. Final check before writing code

- [ ] `git`, `py -3.12`, `docker`, `docker compose`, `node`, `npm` all print versions.
- [ ] Docker Desktop is running and `docker ps` shows `mariadb:10.11` (backend lead).
- [ ] `pip check` says no broken requirements (backend lead).
- [ ] `npm ls --depth=0` and `npm audit` are clean (frontend lead).
- [ ] `.env` exists locally and is in `.gitignore`.
- [ ] Both GitHub repositories exist with a protected `main` branch.
- [ ] You have your own Gmail with an app password (by week 2).
