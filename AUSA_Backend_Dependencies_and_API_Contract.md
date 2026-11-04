# AUSA E-Voting System: Backend Dependencies and API Contract

Version 4.1, November 2, 2026. Matches the Project Blueprint, Draft v4, plus the hosting update below.

**What changed in v4.1:** Docker Compose runs staging and production (Django + Gunicorn, MariaDB 10.11, Caddy for HTTPS). Laptops run only MariaDB in Docker. The frontend and the API share **one address**, with the API under `/api/`, so `django-cors-headers` and the cookie-domain settings are gone. The address is either our own domain (Option A) or a subdomain from the school ICT unit (Option B); see A8. Nothing in Part B's endpoints changed.

**What changed in v4:** sign-up now starts with school email plus matric number and is checked against the school list before any OTP is sent; OTP check and password setting are separate steps; the student logs in with matric number and password after sign-up; the election token pops up once at the first login after the token window opens; students stay logged in after voting and see a Voted status; results appear on every student's dashboard only after the admin publishes them, which is only possible after the election ends; the admin can extend a live election or reopen a closed one (before results are published). OTP email is sent from the school's Google Workspace account.

This file is the agreement between the backend (Django REST API) and the frontend (React). Once both leads sign it off, neither side changes a URL, a field name, a status code or an error code without telling the other. Items marked **CONFIRM** still need a decision from the team or the school.

---

## Part A. Backend dependencies

### A1. Runtime and tools

| Item | Version | Notes |
|---|---|---|
| Python | 3.12 (3.10 to 3.13 work) | Django 5.2 supports 3.10 to 3.13. Use the same minor version on laptop and server. |
| Django | 5.2 LTS (5.2.18 at time of writing) | Long-term support. Do not jump to 6.x mid-project; some add-ons below only list support up to 6.0. |
| MariaDB | 10.11 LTS, in Docker everywhere | Django 5.2 refuses MariaDB 10.4, and **many XAMPP builds ship 10.4**. Do not use XAMPP's database: run the `mariadb:10.11` container from `compose.dev.yaml` on laptops (A3) and from `compose.yaml` on servers (A7). |
| Docker | Docker Engine with Compose v2 on servers; Docker Desktop (WSL2 backend) on Windows laptops | Laptops use Docker only for the database; Django and React run normally there for easy debugging. |
| Caddy | 2.x, in Docker | Serves the React build, sends `/api/*` to Django, serves photos, and gets and renews HTTPS certificates by itself. Replaces Nginx and certbot. |
| Redis | 6 or newer | Only if you use Celery for the email queue. |
| Git | any recent | `.env` must be in `.gitignore`. |

### A2. Python packages (verified to install together, `pip check` clean, `manage.py check` clean)

**requirements.txt** (production)

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

Add only if you use them:

```text
# Only if the live sender is Amazon SES (not needed for Gmail or Google Workspace SMTP)
django-anymail[amazon-ses]==15.2

# Only if you add the email queue
celery==5.6.3
redis==8.1.0
```

**requirements-dev.txt**

```text
-r requirements.txt
pytest==9.1.1
pytest-django==4.14.0
factory-boy==3.3.3
pip-audit==2.10.1
```

| Package | Why we need it |
|---|---|
| Django | Main framework, ORM, sessions, CSRF, password hashing, admin. |
| djangorestframework | Builds the JSON API, serializers, permissions, throttling. |
| mysqlclient | Connects Django to MariaDB. If it will not install on Windows, use `PyMySQL==1.2.3` and add `import pymysql; pymysql.install_as_MySQLdb()` at the top of `config/__init__.py` (laptop only; use mysqlclient on the server). |
| django-environ | Reads all secrets and settings from `.env`. |
| django-axes | Locks repeated wrong logins. Configure the lock on matric number **plus** IP address, so nobody can lock a real student out by typing their matric number. |
| drf-spectacular | Live API docs at `/api/docs/` generated from the code, so this contract can be checked against reality. |
| Pillow | Validates candidate and executive photos. |
| nh3 | Cleans blog HTML on the server. |
| argon2-cffi | Enables Django's Argon2 password hasher (stronger than the default). |
| django-otp | Authenticator-app codes (TOTP) for admin and officer accounts. Free, needs no email or SMS. |
| gunicorn | Runs Django inside the `web` container, behind Caddy. |
| whitenoise | Serves Django's own static files (admin, API docs) under `/api/static/`. |
| django-anymail | Only for Amazon SES. Gmail and Google Workspace use Django's built-in SMTP backend. |
| celery, redis | Optional queue so OTP emails go out in order without slowing requests. |
| pytest, pytest-django, factory-boy | Tests and fake data. |
| pip-audit | Checks packages for known vulnerabilities before each release. |

### A3. Setup on Windows (PowerShell, inside the project folder)

```powershell
py -3.12 -m venv venv
venv\Scripts\activate
python -m pip install --upgrade pip
pip install -r requirements-dev.txt
django-admin startproject config .
python manage.py startapp accounts
python manage.py startapp elections
python manage.py startapp voting
python manage.py startapp content
python manage.py startapp audit

# Database: MariaDB 10.11 in Docker (Docker Desktop must be running)
docker compose -f deploy/compose.dev.yaml up -d
# then in .env: DATABASE_URL=mysql://evote_user:devpassword@127.0.0.1:3306/evote
python manage.py migrate
```

`compose.dev.yaml` (A7) creates the `evote` database with `utf8mb4` and a user that only has rights on it. The database listens on `127.0.0.1` only, so nothing outside the laptop can reach it.

### A4. Settings that must exist (summary)

```python
INSTALLED_APPS += [
    "rest_framework", "axes", "drf_spectacular",
    "django_otp", "django_otp.plugins.otp_totp",
    "accounts", "elections", "voting", "content", "audit",
]
# Order matters: WhiteNoiseMiddleware right after SecurityMiddleware,
# OTPMiddleware right after AuthenticationMiddleware, AxesMiddleware last.
AUTHENTICATION_BACKENDS = [
    "axes.backends.AxesStandaloneBackend",
    "django.contrib.auth.backends.ModelBackend",
]
AXES_FAILURE_LIMIT = 5
AXES_COOLOFF_TIME = 0.25            # hours = 15 minutes
AXES_LOCKOUT_PARAMETERS = [["username", "ip_address"]]
AXES_RESET_ON_SUCCESS = True
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",
]
REST_FRAMEWORK = {
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
    "NUM_PROXIES": 1,                       # Caddy sits in front: throttles read the real client IP
}

# One address: everything Django serves lives under /api/
STATIC_URL = "/api/static/"                # served by whitenoise
STATIC_ROOT = BASE_DIR / "staticfiles"
MEDIA_URL = "/media/"                      # photos, served by Caddy from the shared volume
MEDIA_ROOT = BASE_DIR / "media"
# urls.py: path("api/v1/", ...), path("api/schema/", ...), path("api/docs/", ...),
#          path("api/dj-admin/", admin.site.urls)   # not /admin/: that path belongs to the React admin console

# Behind Caddy
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
AXES_IPWARE_PROXY_COUNT = 1
AXES_IPWARE_META_PRECEDENCE_ORDER = ["HTTP_X_FORWARDED_FOR", "REMOTE_ADDR"]
```

No CORS package or cookie-domain settings are needed, because the browser sees the frontend and the API as the same site. These lines were tested with the package versions above (`manage.py check`, `migrate` and `collectstatic` all pass).

### A5. Email: one setup that can switch providers with no code change

All email goes through Django's mail system. The code calls one function (`send_otp_email`). Which service sends it is decided only by `.env`.

```dotenv
# ---- Development: personal Gmail (testing only, about 500 recipients a day) ----
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=ausa.test@gmail.com
EMAIL_HOST_PASSWORD=<16-character app password>   # needs 2-step verification on the account
EMAIL_TIMEOUT=10
DEFAULT_FROM_EMAIL=AUSA E-Vote <ausa.test@gmail.com>
OTP_DAILY_SEND_LIMIT=450

# ---- Live (agreed with the school ICT unit): school Google Workspace account (up to 2,000 a day) ----
# Same EMAIL_BACKEND, HOST, PORT and TLS as above. Change only:
# EMAIL_HOST_USER=no-reply@<school domain>
# EMAIL_HOST_PASSWORD=<app password for that account>
# DEFAULT_FROM_EMAIL=AUSA E-Vote <no-reply@<school domain>>
# OTP_DAILY_SEND_LIMIT=1900
# If the school does not allow app passwords, their admin can set up the
# Google SMTP relay instead: EMAIL_HOST=smtp-relay.gmail.com, allowed by server IP.

# ---- Fallback only if the school account cannot be used: Amazon SES (about USD 0.10 per 1,000 emails) ----
# EMAIL_BACKEND=anymail.backends.amazon_ses.EmailBackend
# AWS_ACCESS_KEY_ID=...  AWS_SECRET_ACCESS_KEY=...  AWS_REGION=eu-west-1
# DEFAULT_FROM_EMAIL=AUSA E-Vote <no-reply@<our domain>>
# OTP_DAILY_SEND_LIMIT=50000
```

Rules for the backend code so switching stays a settings change:

1. All OTP email is sent from one function, `accounts/emails.py: send_otp_email(student, code, purpose)`, which uses `django.core.mail.send_mail` and an email template. Nowhere else sends mail.
2. No email address, host, password or limit is written in code. They come from `.env`.
3. The daily cap (`OTP_DAILY_SEND_LIMIT`) and the global hourly cap (`OTP_GLOBAL_HOURLY_LIMIT`) are settings. When the cap is reached, the API returns `503 EMAIL_CAPACITY_REACHED` with `retry_after_seconds`, instead of failing silently.
4. If the mail server rejects an address, mark the student's `email_status` as `bounced` so officers can fix it, and do not keep retrying.
5. Switch providers on staging first, test delivery to Gmail, Yahoo and school inboxes, then switch live **before** the activation window opens, never during it.

### A6. Full `.env.example`

```dotenv
DJANGO_SECRET_KEY=change-me
DJANGO_DEBUG=False
# ---- the one public address (A8): Option A or Option B ----
SITE_ADDRESS=vote.yourdomain.com                 # Option B: vote.adelekeuniversity.edu.ng
DJANGO_ALLOWED_HOSTS=vote.yourdomain.com         # same value as SITE_ADDRESS
CSRF_TRUSTED_ORIGINS=https://vote.yourdomain.com # https:// + SITE_ADDRESS
# ---- database (servers: host is the "db" container) ----
DB_NAME=evote
DB_USER=evote_user
DB_PASSWORD=long-random-string
DB_ROOT_PASSWORD=another-long-random-string
DATABASE_URL=mysql://evote_user:long-random-string@db:3306/evote
TIME_ZONE=Africa/Lagos
OTP_HMAC_KEY=long-random-string
TOKEN_HMAC_KEY=different-long-random-string
OTP_DAILY_SEND_LIMIT=450
OTP_GLOBAL_HOURLY_LIMIT=300
ALLOW_ABSTAIN=True
HELPDESK_REISSUE_DURING_LIVE=True
SESSION_COOKIE_AGE=1800
SIGNUP_MISMATCH_LIMIT_PER_HOUR=5
# plus the EMAIL_* block from A5
```

On servers this one `.env` sits in `deploy/`: Compose reads the `DB_*` and `SITE_ADDRESS` values, and the `web` container reads the rest. On laptops, `DATABASE_URL` points at `127.0.0.1` instead of `db`.

### A7. Docker files

These four files go in the backend repository. They were checked with `docker compose config` and `caddy validate`; they have not yet been run end to end, so do that on staging in week 2.

**`Dockerfile`** (repository root)

```dockerfile
# backend/Dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Build tools and MariaDB client headers for mysqlclient
RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential pkg-config default-libmysqlclient-dev \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
RUN useradd --create-home app \
 && mkdir -p /app/media /app/staticfiles \
 && chown -R app:app /app
USER app

EXPOSE 8000
# Collect Django's static files (served by whitenoise), then start Gunicorn.
# Migrations are run on purpose as a separate deploy step, not here.
CMD ["sh", "-c", "python manage.py collectstatic --noinput && gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 3 --timeout 30"]
```

**`deploy/compose.yaml`** (staging and production)

```yaml
# deploy/compose.yaml  (staging and production)
services:
  db:
    image: mariadb:10.11
    restart: unless-stopped
    environment:
      MARIADB_DATABASE: ${DB_NAME}
      MARIADB_USER: ${DB_USER}
      MARIADB_PASSWORD: ${DB_PASSWORD}
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    command: ["--character-set-server=utf8mb4", "--collation-server=utf8mb4_unicode_ci"]
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 10
    # no "ports:" on purpose: the database is never reachable from the internet

  web:
    build: ..
    restart: unless-stopped
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - media:/app/media

  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      SITE_ADDRESS: ${SITE_ADDRESS}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./frontend-dist:/srv/frontend:ro
      - media:/srv/media:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - web

volumes:
  db_data:
  media:
  caddy_data:
  caddy_config:
```

**`deploy/Caddyfile`**

```caddyfile
# deploy/Caddyfile
# SITE_ADDRESS is the one public address, e.g. vote.yourdomain.com (Option A)
# or vote.adelekeuniversity.edu.ng (Option B). Caddy gets and renews HTTPS itself.
{$SITE_ADDRESS} {
	encode gzip

	header {
		Strict-Transport-Security "max-age=31536000; includeSubDomains"
		X-Content-Type-Options "nosniff"
		X-Frame-Options "DENY"
		Referrer-Policy "same-origin"
		Content-Security-Policy "default-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline'; script-src 'self'; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'"
		-Server
	}

	# Django: API, API docs, Django admin and Django's own static files
	handle /api/* {
		reverse_proxy web:8000
	}

	# Candidate and executive photos
	handle /media/* {
		root * /srv
		file_server
	}

	# React app: every other path returns index.html so client routes work
	handle {
		root * /srv/frontend
		try_files {path} /index.html
		file_server
	}
}
```

**`deploy/compose.dev.yaml`** (laptops: database only)

```yaml
# deploy/compose.dev.yaml  (laptops: only the database runs in Docker)
services:
  db:
    image: mariadb:10.11
    environment:
      MARIADB_DATABASE: evote
      MARIADB_USER: evote_user
      MARIADB_PASSWORD: devpassword
      MARIADB_ROOT_PASSWORD: devrootpassword
    command: ["--character-set-server=utf8mb4", "--collation-server=utf8mb4_unicode_ci"]
    ports:
      - "127.0.0.1:3306:3306"
    volumes:
      - dev_db:/var/lib/mysql

volumes:
  dev_db:
```

**Deploying (staging or production)**

```bash
# on the server, once: install Docker Engine + Compose plugin, clone the repo, write deploy/.env
cd deploy
# put the React build in deploy/frontend-dist (npm run build in the frontend repo, then copy dist/)
docker compose up -d --build                       # builds web, starts db, web, caddy
docker compose run --rm web python manage.py migrate
docker compose run --rm web python manage.py createsuperuser   # first deploy only
# later deploys: git pull, refresh frontend-dist, then the same up -d --build and migrate
```

**Daily backup** (a cron job on the host, copied off the server afterwards):

```bash
docker compose -f /path/to/deploy/compose.yaml exec -T db \
  sh -c 'mariadb-dump -u root -p"$MARIADB_ROOT_PASSWORD" --single-transaction "$MARIADB_DATABASE"' \
  | gzip > /backups/evote-$(date +%F).sql.gz
```

### A8. The public address: Option A or Option B

The frontend and the API share **one address**. Caddy sends `/api/*` to Django and everything else to the React app, so the frontend calls `/api/v1/...` on the same address.

| | Option A: our own domain (original plan) | Option B: a subdomain from the school ICT unit |
|---|---|---|
| Address | `vote.yourdomain.com` (staging: `staging.vote.yourdomain.com`) | `vote.adelekeuniversity.edu.ng` (staging: `vote-staging.adelekeuniversity.edu.ng`, if ICT agrees) |
| Cost | About ₦15,000 to ₦30,000 a year | Free |
| Who controls DNS | We do | ICT does; changes go through them |
| Cloudflare in front | Yes, put our domain on Cloudflare's free plan | Only if the school's whole DNS is on Cloudflare; otherwise rely on Caddy and Django rate limits |
| Trust | Students may not recognise the domain | School address, matches the OTP sender |

**Steps, Option A**

1. Buy the domain and add it to Cloudflare (free plan).
2. Rent the servers; on DigitalOcean attach a reserved IP so the address survives a resize.
3. In Cloudflare DNS, add an `A` record `vote` → production IP and `staging.vote` → staging IP.
4. Set `SITE_ADDRESS`, `DJANGO_ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS` in `deploy/.env`, then `docker compose up -d`. Caddy gets the certificate on first start.

**Steps, Option B**

1. Rent the servers and note their public IPs (reserved IPs on DigitalOcean).
2. Ask ICT to add `A` records: `vote.adelekeuniversity.edu.ng` → production IP and `vote-staging.adelekeuniversity.edu.ng` → staging IP, with a TTL of 300 seconds. Give them a contact name and phone number.
3. Check: `nslookup vote.adelekeuniversity.edu.ng` returns the server IP.
4. Set `SITE_ADDRESS=vote.adelekeuniversity.edu.ng` (and the two values that follow it) in `deploy/.env`, then `docker compose up -d`. Ports 80 and 443 must be open; Caddy gets the certificate on first start.
5. If ICT cannot give a staging subdomain, use a cheap domain of our own or a Cloudflare Tunnel address for staging. Only production needs the school name.

**Either way:** set the address before the sign-up window opens and never change servers or DNS during sign-up or voting. Use the same domain for the OTP sender where possible (`no-reply-evote@` the school domain in Option B).

---

## Part B. API contract

### B1. Rules for both sides

- Base URL: `/api/v1/` on the **same address as the frontend** (A8), so requests carry the session cookie with no CORS setup. Live docs: `/api/schema/` (OpenAPI) and `/api/docs/` (Swagger UI), turned off in production or protected by staff login. Django's own admin is at `/api/dj-admin/` (staff only); `/admin` is the React admin console.
- JSON everywhere, except image uploads (multipart form data).
- Times are ISO 8601 in UTC, for example `2026-12-01T08:00:00Z`. The frontend converts to Lagos time for display.
- Field names are `snake_case`.
- Login uses a Django session cookie (HttpOnly, Secure, SameSite=Lax). React sends requests with credentials and sends the CSRF cookie value in the `X-CSRFToken` header on every POST, PATCH and DELETE. Nothing secret goes in localStorage.
- Lists are paginated: `?page=1&page_size=20` and return `count`, `next`, `previous`, `results`. Max `page_size` is 100.
- Countdowns use `server_time` from the API, never the phone clock.
- Any response containing an election token is sent with `Cache-Control: no-store`, and the token is never logged.
- Rate limits return `429 RATE_LIMITED` with `retry_after_seconds`.

### B2. Response envelope

```json
{ "success": true, "data": { } }
```

```json
{ "success": false,
  "error": { "code": "OTP_EXPIRED",
             "message": "This code has expired. Request a new one.",
             "fields": { "otp": ["This code has expired."] },
             "retry_after_seconds": null } }
```

`fields` appears only for validation errors. `retry_after_seconds` appears only for 423, 429 and 503.

### B3. Error codes

| HTTP | Code | Meaning |
|---|---|---|
| 400 | VALIDATION_ERROR | A field is missing or badly formatted (see `fields`). Also used for a weak new password. |
| 400 | OTP_INVALID | Wrong code. |
| 400 | OTP_EXPIRED | Code older than 5 minutes, or replaced by a newer code. |
| 400 | DETAILS_NOT_MATCHED | Sign-up or password reset: the email and matric number do not belong to the same student on the school list. Message: "We couldn't verify these details. Check your matric number and school email, or visit the AUSA help desk." Never says which field was wrong. |
| 400 | INVALID_CREDENTIALS | Wrong matric number or password (same message for both). |
| 400 | INVALID_SCHEDULE_CHANGE | Extend or reopen with a new end time that is not later than both the current end time and now. |
| 400 | TOTP_REQUIRED / TOTP_INVALID | Staff login without, or with a wrong, authenticator code. |
| 400 | INVALID_SELECTION | Ballot has a wrong candidate, a missing post, a post the student cannot vote for, or two choices for one post. |
| 401 | NOT_AUTHENTICATED | Not logged in, or the session ended. |
| 403 | FORBIDDEN | Logged in but not allowed (for example a student on an `/admin` URL). |
| 403 | ACCOUNT_NOT_ACTIVE | Account not activated yet. |
| 403 | ACTIVATION_CLOSED | Activation is closed because an election is live. |
| 403 | TOKEN_WINDOW_CLOSED | Tokens cannot be shown or reissued now. |
| 403 | BALLOT_SESSION_REQUIRED | Ballot or vote called without a valid token check in the last 20 minutes. |
| 403 | STEP_EXPIRED | The set-password step was reached without a verified OTP in the last 10 minutes. Start sign-up (or reset) again. |
| 403 | RESULTS_NOT_PUBLISHED | Results asked for before they are published. |
| 404 | NOT_FOUND | Item does not exist. |
| 409 | ACCOUNT_ALREADY_ACTIVE | Already activated; use login or password reset. |
| 409 | TOKEN_ALREADY_SHOWN | The token for this election was already shown once. Use reissue. |
| 409 | TOKEN_USED | The token was already used to vote. |
| 409 | ALREADY_VOTED | This student already voted in this election. |
| 409 | ELECTION_STILL_LIVE | Results cannot be viewed by officers or published while the election is live. |
| 409 | RESULTS_ALREADY_PUBLISHED | The election cannot be extended or reopened after its results are published. |
| 422 | TOKEN_INVALID | Token is wrong or revoked. |
| 422 | ELECTION_NOT_LIVE | Election has not started, or has ended. |
| 423 | OTP_LOCKED | 5 wrong codes; locked for 15 minutes. |
| 423 | SIGNUP_LOCKED | 5 unmatched sign-up or reset attempts from this device in an hour; locked for 1 hour. |
| 423 | TOKEN_LOCKED | 5 wrong tokens; locked for 15 minutes. |
| 423 | ACCOUNT_LOCKED | Too many wrong passwords from this device; locked for 15 minutes. |
| 429 | RATE_LIMITED | Too many requests, or Resend pressed before 60 seconds. |
| 503 | EMAIL_CAPACITY_REACHED | Daily or hourly email limit reached. Try again later. |

### B4. Fixed rules the backend enforces

| Rule | Value |
|---|---|
| Sign-up match | The typed school email (compared lowercase) and matric number (compared after trimming spaces and uppercasing) must belong to the **same** row in the student list. Only then is an OTP sent. |
| Mismatch | Return `DETAILS_NOT_MATCHED` with the same message whatever was wrong. Record a `SignupAttempt` (typed matric, a hash of the typed email, IP, time, outcome) for officers to review. **Never** lock, mark or flag the real student's account because of a mismatch, otherwise anyone could get a classmate flagged. 5 mismatches from one IP in an hour → `SIGNUP_LOCKED` for 1 hour. |
| Already active | A matching email and matric that is already activated returns `ACCOUNT_ALREADY_ACTIVE` ("Your account is already set up. Log in or reset your password."). No OTP is sent. |
| OTP | 6 digits from `secrets`, valid 300 s, one use, stored as HMAC hash. A new code cancels the previous unused code of the same purpose. |
| OTP resend | Not before 60 s. Max 5 OTP emails per student per day. |
| OTP attempts | 5 wrong tries, then `OTP_LOCKED` for 15 minutes. |
| Set-password step | A correct OTP stores a "verified" marker in the server session, valid 10 minutes. The set-password call needs it, otherwise `STEP_EXPIRED`. The matric number comes from the session, not from the request body. |
| After sign-up | The student is **not** logged in automatically. The frontend shows the welcome message and sends them to the login page. |
| Login | Students: matric number plus password. Staff: email, password and authenticator code. |
| Session | HttpOnly cookie, ends after 30 minutes without activity (`SESSION_COOKIE_AGE=1800`, `SESSION_SAVE_EVERY_REQUEST=True`) and when the browser is closed. Voting does **not** log the student out. |
| Activation lock | Sign-up is refused with `ACTIVATION_CLOSED` while an election is live (from `start_at` to the current `end_at`, including any extension or reopening). Password reset stays open. |
| Election token | 12 characters from `ABCDEFGHJKLMNPQRSTUVWXYZ23456789`, shown as `XXXX-XXXX-XXXX`. Input is normalized (uppercase, dashes and spaces removed). Stored only as HMAC-SHA256 with `TOKEN_HMAC_KEY`; compared with `hmac.compare_digest`. Never emailed. |
| Token pop-up | Shown **once**, at the first login after the token window opens (or straight away if the student is already logged in when it opens). After it is revealed it can never be shown again; a lost token is replaced by reissue. |
| Token window | From `token_window_opens_at` until `start_at`. |
| Token reissue | Self-service with a fresh OTP until `start_at`. During voting only the help desk can reissue (if `HELPDESK_REISSUE_DURING_LIVE=True`), and never for a student who has voted. The old token is revoked. |
| Token checks | 5 wrong tokens per student per election, then `TOKEN_LOCKED` for 15 minutes. |
| Ballot session | 20 minutes after a successful token check, stored in the server session. Ends when the vote is submitted. |
| After voting | The student stays logged in. Their dashboard shows **Voted**, the hour they voted and their receipt number, and the vote button is disabled. |
| Password | Django validators, minimum 8 characters, not a common password, not all numbers, not similar to matric number. `new_password` and `confirm_password` must match. |
| Eligible voters | Students with `activated_at` before the election's original `start_at`. Position scope (faculty or department) limits which posts each student sees. |
| Abstain | Allowed when `ALLOW_ABSTAIN=True` (`candidate_id: null`). CONFIRM with the school. |
| Results | Per-candidate counts are hidden from everyone while the election is live. Officers can view counts only after the current `end_at`. The admin can publish only after the current `end_at`. Students see results on their dashboard only after publishing. |
| Extend | Admin only. While the election is live, `end_at` can be moved later (never earlier). A reason is required. Audit-logged with old and new times. |
| Reopen | Admin only. After the election has ended **and before results are published**, it can be reopened with a new `end_at` in the future. Votes already cast stay counted and those students still show Voted. Tokens, eligibility and the activation lock carry on as before. The audit log records the reason and whether any officer viewed results before the reopening. |
| After publishing | Extend and reopen are refused with `RESULTS_ALREADY_PUBLISHED`. Results cannot be unpublished. |

### B5. Endpoints

#### Sign-up, login and password reset (prefix `/auth`)

| Method | URL | Who | Request body | Success |
|---|---|---|---|---|
| GET | `/auth/csrf` | Public | none | 200 `{ "detail": "ok" }` and sets the `csrftoken` cookie. Call once when the app loads. |
| POST | `/auth/signup/start` | Public | `{ "email": "ada.okafor@<school domain>", "matric_no": "20/0123" }` | 200 `{ "message": "We sent a 6-digit code to your school email.", "email_hint": "a***r@<school domain>", "resend_after_seconds": 60, "expires_in_seconds": 300 }` |
| POST | `/auth/signup/verify` | Public | `{ "matric_no": "20/0123", "otp": "482913" }` | 200 `{ "matric_no": "20/0123", "full_name": "Ada Okafor", "step_expires_in_seconds": 600 }` and the session is marked verified for this student. |
| POST | `/auth/signup/complete` | Public with a verified step | `{ "new_password": "...", "confirm_password": "..." }` | 201 `{ "message": "Thank you for coming on board! Your account is ready. Log in with your matric number and password." }`. Not logged in. |
| POST | `/auth/login` | Public (students) | `{ "matric_no": "20/0123", "password": "..." }` | 200 `{ "user": {UserMe} }` |
| POST | `/auth/staff/login` | Public (staff) | `{ "email": "officer@...", "password": "...", "totp_code": "123456" }` | 200 `{ "user": {UserMe} }` |
| POST | `/auth/logout` | Logged in | none | 200 `{ "message": "Signed out." }` |
| GET | `/auth/me` | Logged in | none | 200 `{UserMe}` |
| POST | `/auth/password/forgot` | Public | `{ "email": "...", "matric_no": "..." }` | Same as `signup/start` (account must already be active). |
| POST | `/auth/password/verify` | Public | `{ "matric_no": "...", "otp": "..." }` | Same as `signup/verify`. |
| POST | `/auth/password/reset` | Public with a verified step | `{ "new_password": "...", "confirm_password": "..." }` | 200 `{ "message": "Password updated. Log in with your new password." }` |

To resend a code, call the matching `start` or `forgot` endpoint again after 60 seconds.

`UserMe` shape:

```json
{ "id": 41, "role": "student", "full_name": "Ada Okafor",
  "matric_no": "20/0123", "faculty": "Science", "department": "Computer Science", "level": 300,
  "current_election": {
    "id": 3,
    "token_status": "available",
    "voting_status": "not_open",
    "voted_hour": null,
    "receipt_id": null } }
```

- `role`: `student`, `officer` or `admin`. Staff have `matric_no`, `faculty`, `department`, `level` and `current_election` as `null`.
- `current_election` is `null` when there is no scheduled, live or closed election to show.
- `token_status`: `not_open` (window not open yet), `available` (frontend must call `token/reveal` now and show the pop-up), `shown`, `used`, `closed` (window closed and never shown).
- `voting_status`: `not_eligible` (activated after the start), `not_open` (before start), `ready` (live, not voted), `voted`, `closed` (ended, did not vote).
- `voted_hour` and `receipt_id` are filled once the student has voted.

Errors: `signup/start` and `password/forgot` → `VALIDATION_ERROR`, `DETAILS_NOT_MATCHED`, `ACCOUNT_ALREADY_ACTIVE` (signup), `ACCOUNT_NOT_ACTIVE` (forgot), `ACTIVATION_CLOSED` (signup), `SIGNUP_LOCKED`, `RATE_LIMITED`, `EMAIL_CAPACITY_REACHED`. `signup/verify` and `password/verify` → `OTP_INVALID`, `OTP_EXPIRED`, `OTP_LOCKED`. `signup/complete` and `password/reset` → `STEP_EXPIRED`, `VALIDATION_ERROR` (weak password, or `confirm_password` does not match), `ACTIVATION_CLOSED` (signup). `login` → `INVALID_CREDENTIALS`, `ACCOUNT_NOT_ACTIVE`, `ACCOUNT_LOCKED`. `staff/login` → `INVALID_CREDENTIALS`, `TOTP_REQUIRED`, `TOTP_INVALID`, `ACCOUNT_LOCKED`.

#### Elections, tokens, voting and results (students)

| Method | URL | Who | Request body | Success |
|---|---|---|---|---|
| GET | `/elections/current` | Public (extra fields when logged in) | none | 200 `{CurrentElection}` or `null` |
| POST | `/elections/{id}/token/reveal` | Student, `token_status = available` | none | 200 `{TokenReveal}`. Works once; afterwards `TOKEN_ALREADY_SHOWN`. |
| POST | `/elections/{id}/token/reissue/start` | Student | none | 200 `{ "message": "We sent a code to your school email.", "resend_after_seconds": 60, "expires_in_seconds": 300 }` |
| POST | `/elections/{id}/token/reissue/complete` | Student | `{ "otp": "551204" }` | 200 `{TokenReveal}`; the old token is revoked. |
| POST | `/elections/{id}/token/verify` | Student | `{ "token": "K7M2-9QXA-4TPD" }` | 200 `{ "ballot_session_expires_in_seconds": 1200 }` |
| GET | `/elections/{id}/ballot` | Student with ballot session | none | 200 `{Ballot}` |
| POST | `/elections/{id}/votes` | Student with ballot session | `{ "selections": [ { "position_id": 11, "candidate_id": 101 }, { "position_id": 12, "candidate_id": null } ] }` | 201 `{ "message": "Your vote has been recorded.", "receipt_id": "R-8F3A21", "voted_hour": "2026-12-01T09:00:00Z" }`. The ballot session ends; the login session stays. |
| GET | `/elections/{id}/results` | Logged in | none | 200 `{Results}` |

`TokenReveal` shape:

```json
{ "election_id": 3, "token": "K7M2-9QXA-4TPD", "shown_once": true }
```

`CurrentElection` shape:

```json
{ "id": 3, "title": "AUSA General Election 2026",
  "status": "live",
  "token_window_opens_at": "2026-11-17T08:00:00Z",
  "start_at": "2026-12-01T08:00:00Z",
  "end_at": "2026-12-01T19:00:00Z",
  "original_end_at": "2026-12-01T18:00:00Z",
  "times_extended": 1, "reopened": false,
  "server_time": "2026-12-01T09:14:22Z",
  "activation_open": false, "token_window_open": false,
  "turnout": { "voted": 3120, "total": 9874 },
  "results_published": false }
```

- `status` is `scheduled`, `live` or `closed` (drafts are never returned). A reopened election is `live` again until its new `end_at`.
- `end_at` is always the current closing time. When it differs from `original_end_at`, the frontend shows "Voting extended to …".
- `turnout.total` is the number of eligible voters.
- The dashboard and ballot pages poll this endpoint every 60 seconds so countdowns pick up an extension or a reopening without a reload.

`Ballot` shape (posts filtered to the student's faculty and department; candidate order shuffled per request):

```json
{ "election_id": 3, "allow_abstain": true,
  "positions": [
    { "id": 11, "title": "President",
      "candidates": [
        { "id": 101, "full_name": "A. Student", "faculty": "Science",
          "department": "Computer Science", "level": 400,
          "photo_url": "/media/candidates/9f2c.jpg", "manifesto": "Plain text..." } ] } ] }
```

`Results` shape (available to students only when `results_published` is true):

```json
{ "election_id": 3, "published_at": "2026-12-01T19:30:00Z",
  "turnout": { "voted": 6203, "total": 9874 },
  "positions": [
    { "id": 11, "title": "President", "abstain": 120,
      "candidates": [
        { "id": 101, "full_name": "A. Student", "votes": 3226, "is_winner": true } ] } ] }
```

Errors: reveal → `TOKEN_WINDOW_CLOSED`, `TOKEN_ALREADY_SHOWN`. Reissue start and complete → `TOKEN_WINDOW_CLOSED`, `RATE_LIMITED`, `EMAIL_CAPACITY_REACHED`, `OTP_INVALID`, `OTP_EXPIRED`, `OTP_LOCKED`. Verify → `ELECTION_NOT_LIVE`, `TOKEN_INVALID`, `TOKEN_USED`, `ALREADY_VOTED`, `TOKEN_LOCKED`. Ballot → `BALLOT_SESSION_REQUIRED`, `ELECTION_NOT_LIVE`, `ALREADY_VOTED`. Votes → `BALLOT_SESSION_REQUIRED`, `INVALID_SELECTION`, `ALREADY_VOTED`, `TOKEN_USED`, `ELECTION_NOT_LIVE`. Results → `RESULTS_NOT_PUBLISHED`, `NOT_FOUND`.

**Vote submit is one database transaction:** lock the student's token row (`select_for_update`), check the election is live by server time against the current `end_at`, check the token is unused and not revoked, validate selections (exactly one entry per eligible post, candidate belongs to that post, abstain only if allowed), create the `VoteRecord` (unique on election and student), create the `Ballot` rows (random UUIDs, shuffled order, no student, no time), mark the token used. Any failure saves nothing. A unique-constraint error is returned as `ALREADY_VOTED`.

#### Student dashboard states (frontend, from `/auth/me` and `/elections/current`)

| State | What the dashboard shows |
|---|---|
| `not_open` | Election name, countdown to the start, token status ("Saved" or "Get your token"). |
| `ready` | Countdown to the close, "Vote now" button. |
| `voted` | **Voted** badge, hour voted, receipt number. "Vote now" is disabled. Turnout bar. |
| `closed`, results not published | "Voting has closed. Results will appear here when they are published." |
| results published | Results card for every post: candidates, votes, winner, abstentions, turnout. Same for every student. |
| `not_eligible` | "You activated after voting started, so you cannot vote in this election." |

#### Admin and officer (prefix `/admin`; staff session with a verified TOTP code)

| Method | URL | Who | Purpose |
|---|---|---|---|
| GET, POST | `/admin/elections` | Officer | List or create (`title`). New elections start as `draft`. |
| GET, PATCH, DELETE | `/admin/elections/{id}` | Officer | View or edit. Delete only while `draft`. Times, positions and candidates lock at `start_at`; after that only extend and reopen change times. |
| POST | `/admin/elections/{id}/schedule` | Officer | `{ "token_window_opens_at", "start_at", "end_at" }`. Must be in order, in the future, and the window must open at least 24 hours before start (14 days recommended). Moves `draft` to `scheduled`. Sets `original_end_at`. |
| POST | `/admin/elections/{id}/extend` | Admin | `{ "new_end_at", "reason" }`. Only while live. `new_end_at` must be later than the current `end_at`. Errors: `ELECTION_NOT_LIVE`, `INVALID_SCHEDULE_CHANGE`, `RESULTS_ALREADY_PUBLISHED`. |
| POST | `/admin/elections/{id}/reopen` | Admin | `{ "new_end_at", "reason" }`. Only after the current `end_at` and before results are published. `new_end_at` must be in the future. Response includes `results_viewed_before_reopen` (true or false). Errors: `ELECTION_STILL_LIVE`, `INVALID_SCHEDULE_CHANGE`, `RESULTS_ALREADY_PUBLISHED`. |
| POST | `/admin/elections/{id}/positions` | Officer | `{ "title", "display_order", "scope": "all" or "faculty" or "department", "scope_value" }`. PATCH or DELETE at `/admin/positions/{id}`. |
| POST | `/admin/positions/{id}/candidates` | Officer | Multipart: `full_name`, `matric_no`, `faculty`, `department`, `level`, `manifesto` (plain text, max 1,000 characters), `photo` (JPG or PNG, max 2 MB). PATCH or DELETE at `/admin/candidates/{id}`. |
| GET | `/admin/elections/{id}/token-status` | Officer | Counts only: `activated`, `token_shown`, `not_shown`, `reissued`, `helpdesk_reissued`. Never token values. |
| POST | `/admin/elections/{id}/tokens/reissue` | Officer (help desk) | `{ "matric_no", "id_card_checked": true, "reason" }`. Revokes the old token and returns `{TokenReveal}` once, to show the student at the desk. Refused if the student has voted. During voting only if `HELPDESK_REISSUE_DURING_LIVE=True` (CONFIRM). Always audit-logged. |
| GET | `/admin/elections/{id}/turnout` | Officer | Voted over eligible, with a faculty breakdown. No candidate counts. |
| GET | `/admin/elections/{id}/results` | Officer | Full counts, only after the current `end_at`, otherwise `ELECTION_STILL_LIVE`. Every view is audit-logged. |
| POST | `/admin/elections/{id}/publish-results` | Admin | Makes results visible on every student's dashboard. Only after the current `end_at`, otherwise `ELECTION_STILL_LIVE`. Cannot be undone. |
| GET | `/admin/signup-attempts` | Officer | Unmatched sign-up and reset attempts (typed matric, time, IP, count), for the help desk to spot typos and abuse. |
| POST | `/admin/students/import?dry_run=true` | Admin | Multipart CSV with columns `matric_no, email, full_name, faculty, department, level`. Returns `{ "rows_read", "valid", "duplicates", "errors": [ { "row", "field", "message" } ] }`. Call again with `dry_run=false` to save. |
| GET | `/admin/students` | Officer | Filters: `search`, `faculty`, `department`, `level`, `activated`, `email_status`, `voted` (for an `election_id`). |
| PATCH | `/admin/students/{id}` | Admin | Correct `email` (and reset `email_status`). Requires `reason`. Audit-logged. |
| POST | `/admin/staff` | Admin | Create an officer: `{ "email", "full_name" }`. They set a password and TOTP on first login. |
| GET | `/admin/audit-logs` | Admin | Filter by `actor`, `action`, date range. Read only. |

#### Public content

| Method | URL | Purpose |
|---|---|---|
| GET | `/posts`, `/posts/{slug}` | Published blog and event posts only. |
| GET | `/administrations`, `/administrations/current` | Administrations and the current one with executives. Empty until the admin adds one. |
| GET | `/executives` | Executives of the current administration. |
| POST, PATCH, DELETE | `/admin/posts`, `/admin/administrations`, `/admin/executives` | Officer manages content. Post bodies are cleaned with nh3 before saving. |

### B6. Data model (summary)

| Table | Key fields | Rules |
|---|---|---|
| User | email (staff login), role (`admin`, `officer`, `student`), password hash, is_active | Django custom user, created before the first migration. |
| Student | user (one to one, empty until sign-up is complete), matric_no (unique), email (unique, lowercase), full_name, faculty, department, level, is_activated, activated_at, email_status (`ok`, `bounced`) | Loaded from the school CSV. |
| OTP | student, purpose (`signup`, `password_reset`, `token_reissue`), code_hash, expires_at, attempts, used_at, created_at | Never the raw code. |
| SignupAttempt | matric_typed, email_typed_hash, ip, purpose, outcome (`matched`, `not_matched`, `already_active`), created_at | For review only. Deleted after the election (CONFIRM retention). Never changes the student record. |
| Election | title, token_window_opens_at, start_at, end_at (current), original_end_at, times_extended, reopened_at, status, results_published_at, results_first_viewed_at, created_by | Status comes from times plus admin actions. |
| Position | election, title, display_order, scope, scope_value | |
| Candidate | position, full_name, matric_no, faculty, department, level, photo, manifesto | |
| ElectionToken | election, student, token_hash, issued_at, shown_at, revoked_at, issued_by (`self`, `self_reissue`, `helpdesk`), issued_by_user, used_at, failed_attempts, locked_until | At most one non-revoked token per student per election. |
| VoteRecord | election, student, voted_hour (rounded to the hour), receipt_id | Unique on (election, student). No choices. |
| Ballot | id (UUID), election, position, candidate (empty = abstain) | No student, no time, no sequential id. |
| Post, Administration, Executive | as in the blueprint | |
| AuditLog | actor, action, target_type, target_id, ip, metadata (JSON), created_at | Insert only. Extend and reopen store old end time, new end time and reason. Never contains OTPs, tokens or ballot choices. |
