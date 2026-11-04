# AUSA E-Voting System: Milestones, Phases and Tasks

Version 1, November 2, 2026. Built from Blueprint v4, the Backend Dependencies and API Contract v4.1, the Backend Agent Prompt v4.1 and the Full Project Roadmap.

**What this is.** The election is for the **Adeleke University Students Association (AUSA)**. The **NACOS team** is building and deploying it. This file lists every task, in order, so nobody is ever stuck asking "what is next?"

**How to use it.**
1. Find the first unticked box in the earliest phase that is not finished. That is your next task.
2. A task is only ticked when its **Done means** line is true, not when the code is written.
3. If a task is blocked, write the blocker in the Blocker log (end of file) and move to the next task that is not blocked.

**Team.** Backend lead (builds the Django API), Frontend lead (builds the React app), 2 pen testers. Every task below is tagged **[B]** backend, **[F]** frontend, **[P]** pen testers, **[ALL]** everyone, or **[SCHOOL]** needs the school.

---

## Section 1. Things to note before doing ANY work

Read these once, together. Most expensive mistakes on this project are on this list.

### 1.1 Rules about the documents

1. **If documents disagree, this order wins:** the API Contract file, then the Backend Agent Prompt, then Blueprint v4, then the Roadmap PDF. The contract file says it wins over Blueprint v4.
2. **Known differences already found:**
   - Blueprint v4 still mentions Nginx, certbot and `django-cors-headers`. The contract (v4.1) uses **Caddy in Docker** and **no CORS package**, because the React app and the API share one address. Follow the contract.
   - Blueprint v4 says about 8 weeks. The Roadmap says about **10 weeks** (7 building and testing, then a 2 to 3 week sign-up window). Plan on 10.
   - The sample address `ausa.test@gmail.com` in the email settings is a placeholder. Use your own Gmail.
3. **Nobody changes a URL, a field name, a status code or an error code without telling the other lead.** If you find a problem in the contract, raise it, agree, then both update.
4. **Anything marked CONFIRM is not decided.** Build it behind its setting, use the default, and list it in the README as pending.

### 1.2 Rules about the date

5. **Set a tentative election date in week 0.** Work every other date back from it. Coding must start at least 10 weeks before election day.
6. **Check the date against today.** If the election is meant to happen within about 8 weeks from today (November 2, 2026), the 10 week plan does not fit. Either move the date now, or cut scope now (the cut list is in 1.7), not in week 5.
7. **Sign-up opens at least 14 days before the election.** The school email account can send about 2,000 a day, so 12,000 students need at least 8 to 10 days.

### 1.3 Rules about tools and setup

8. **Never use XAMPP's database.** Many XAMPP builds ship MariaDB 10.4, which Django 5.2 refuses. Use MariaDB 10.11 in Docker.
9. **Stay on Django 5.2 LTS and the pinned versions.** Do not upgrade to Django 6.x or change a pinned version without asking the other lead.
10. **Create the custom User model before the first migration** (`AUTH_USER_MODEL = "accounts.User"`). Changing it later is very painful.
11. **Docker Desktop must be running** before any `docker` or `migrate` command.
12. **Windows commands only on laptops** (PowerShell). Server commands are Linux because the servers run Ubuntu.
13. **Do not install `django-cors-headers`** and do not set cookie-domain settings.

### 1.4 Rules about security and secrets

14. **Never commit `.env`, real student data, or any secret.** Add `.env` to `.gitignore` before the first commit.
15. **Three different random keys:** `DJANGO_SECRET_KEY`, `OTP_HMAC_KEY`, `TOKEN_HMAC_KEY`. Never reuse one for another.
16. **Real student data goes only on production**, never on laptops or staging. Use fake data (`seed_fake_data`) everywhere else.
17. **Never email an election token.** Never store a raw OTP or token. Never log OTP or token requests or responses. Never put OTPs, tokens or vote choices in the audit log.
18. **The server decides time, role, eligibility and vote rules.** Never trust the frontend for these. All "is the election live" checks use the server clock.
19. **All OTP email is sent from one function:** `accounts/emails.py: send_otp_email`. Nothing else sends email.
20. **Mismatches must never flag or lock a real student.** If they could, anyone could get a classmate flagged by typing their matric number with a wrong email.

### 1.5 Rules about the vote itself

21. **Ballot secrecy:** the `Ballot` table has no link to a student, no timestamp, no sequential ID, and rows use random UUIDs inserted in shuffled order. `VoteRecord` keeps only the **hour** of voting.
22. **No per-candidate counts while voting is live**, for anyone. Officers see counts only after the current end time. Results are published only after the end.
23. **Voting does not log the student out.** The dashboard shows Voted, the hour and a receipt number, and the Vote button is disabled.
24. **Extend and reopen:** admin only, reason required, logged. Extend only moves the end time later. Neither is allowed after results are published.

### 1.6 Rules about people and the school

25. **Get the signed agreement in week 0**, and always before real data arrives. Worst case, sign it before money is spent on a domain and server (week 2).
26. **Get the data-sharing agreement before the real student list arrives.** It says how the data is sent, how long it is kept, and when it is deleted from every laptop. Student data is personal data, so follow Nigeria's data protection law (NDPA).
27. **Ask ICT for the Workspace email account in week 0.** Decide by the end of week 5. If it has not arrived by then, start Amazon SES (about a week for DNS records and approval).
28. **Three things decide the election date:** the date itself (week 0), the school email account (end of week 5) and the real student list with its agreement (start of week 8). If one slips, the election moves.
29. **Help desk plan:** lost tokens on election day need an officer, an ID card check, and an audit log entry. Train the officers before election week.

### 1.7 Cut list if time runs short

Cut in this order, and never cut anything above the line:

- Never cut: sign-up, login, tokens, voting, results, security tests, backups.
- Cut first: blog, executives and administration pages.
- Cut second: voter-roll screens polish, faculty breakdown on results.

### 1.8 Working habits

30. **Branches and reviews:** `main` is protected. Every change is a pull request reviewed by the **other** lead.
31. **Module order for the backend:** auth first, then elections, then tokens and voting, then content. Each module is deployed to staging and the frontend lead is told it is ready.
32. **After each backend step, stop and show the result** before starting the next (the agent prompt requires this).
33. **Tests are part of the work.** A step is not done until its tests pass.
34. **Freeze rule:** no server or DNS changes during sign-up or voting.

---

## Section 2. Milestones at a glance

Plan on about 10 weeks from the first line of code to election day. Fill in the dates once the election date is fixed.

| Milestone | Name | Week | Date | Finished when |
|---|---|---|---|---|
| M0 | Groundwork done | 0 | ____ | Contract signed off, repos created, Docker database running, election date set |
| M1 | Backend foundation | 1 | ____ | Skeleton, models, migrations, CSV import and 12,000 fake students work on staging |
| M2 | Sign-up and login working | 2 | ____ | All `/auth` endpoints pass tests and are on staging; staging server is up |
| M3 | Elections ready | 3 | ____ | Officers can create and schedule an election with posts and candidates |
| M4 | Voting works end to end | 4 | ____ | A student can get a token, verify it, vote once, and see Voted |
| M5 | Feature complete | 5 | ____ | Results, publish, audit log, content pages done; pen test round 1 runs |
| M6 | Mock election passed | 6 | ____ | 30 to 50 volunteers vote and counts match exactly |
| M7 | Code freeze and production live | 7 to 8 | ____ | Load test and pen test round 2 pass; production is up |
| M8 | Sign-up window | 8 to 10 | ____ | Real data in; students activating accounts; help desk running |
| M9 | Election day | 10 | ____ | Voting opens and closes cleanly; results published |
| M10 | Handover | 11 | ____ | Backups kept, report sent, run guide and access handed over |

---

## Section 3. Phase 0: Groundwork (week 0)

These take days of waiting on other people, so start them now, alongside week 1 of code.

### 3.1 Three tasks that truly block coding (about a day in total)

- [ ] **P0.1 [B][F]** Both leads read the contract (Blueprint Section 5 and the contract file Part B) and **sign off API contract v4.1**. *Done means:* both leads have said "agreed" in writing.
- [ ] **P0.2 [ALL]** Create two GitHub repositories (`backend`, `frontend`), protect `main`, require a pull request review from the other lead. *Done means:* a test pull request is blocked until the other lead approves.
- [ ] **P0.3 [B]** Install Docker Desktop and run MariaDB 10.11 in Docker (see `AUSA_Dependencies_to_Install.md`). *Done means:* `docker ps` shows `mariadb:10.11`.

### 3.2 Tools and accounts

- [ ] **P0.4 [ALL]** Install everything in `AUSA_Dependencies_to_Install.md` Part 2 and run the check commands. *Done means:* every check prints a version.
- [ ] **P0.5 [B]** Open the accounts in Part 1 of the install file that are needed in week 0 (GitHub, Cloudflare).
- [ ] **P0.6 [B]** Make a personal Gmail with 2-step verification and an app password for development email (latest safe point: start of week 2; until then Django can print emails to the console).
- [ ] **P0.7 [ALL]** Set up a shared task board and copy the tasks from Phases 1 and 2 into it.

### 3.3 With the school

- [ ] **P0.8 [SCHOOL]** Signed agreement: scope, price, payment schedule, who owns the system and the data after handover. *Done means:* signed by both sides.
- [ ] **P0.9 [SCHOOL]** Data-sharing agreement: fields (matric number, full name, school email, faculty, department, level), how the file is sent securely, how long it is kept, when it is deleted. *Latest safe point:* before the real CSV arrives (week 8, day 1).
- [ ] **P0.10 [SCHOOL]** Ask ICT for a **dedicated** Google Workspace account (for example `no-reply-evote@` on the school domain) with an app password or SMTP relay. In the same request, ask whether ICT will create the subdomain `vote.adelekeuniversity.edu.ng` (Option B). *Decide by end of week 5; hard limit week 7.*
- [ ] **P0.11 [SCHOOL]** Get a **sample CSV** of 10 to 20 rows (fake values, real format). *Latest safe point:* week 3. Without it, the import is built on a guess.
- [ ] **P0.12 [SCHOOL]** Fix a **tentative election date**. Every other date is worked back from it.
- [ ] **P0.13 [SCHOOL]** Ask for answers to the open questions in Section 12. Some are needed sooner than others (see the deadlines there).

### 3.4 Choices to make in week 0 or 1

- [ ] **P0.14 [B]** Choose the public address: **Option A** (own domain, about ₦15,000 to ₦30,000 a year, we control DNS) or **Option B** (school subdomain, free, ICT makes DNS changes, matches the OTP sender). *Latest safe point:* start of week 2.
- [ ] **P0.15 [B]** Choose the server provider: DigitalOcean (dollars) or Lineserve (naira). Use one provider for both servers. *Latest safe point:* week 3 for staging.

---

## Section 4. Phase 1: Backend build (weeks 1 to 7) [B]

The backend lead builds in this order so each module is on staging before the frontend needs it. Stop and show the result after each step.

### Step 1 (week 1): Skeleton

- [ ] **B1.1** Create the project on Python 3.12 and Django 5.2 LTS with settings split into `base.py`, `dev.py`, `prod.py`.
- [ ] **B1.2** Create `.env.example` (committed) and `.env` (not committed). Confirm `.env` is in `.gitignore`.
- [ ] **B1.3** Create the **custom User model first**, then the first migration.
- [ ] **B1.4** Build the response envelope renderer and the exception handler that maps every error to a code from the contract.
- [ ] **B1.5** Install drf-spectacular and confirm the docs page loads at `/api/docs/`.
- [ ] **B1.6** Add the Dockerfile and the `deploy/` files (`compose.yaml`, `compose.dev.yaml`, `Caddyfile`) from contract section A7.
- [ ] **B1.7** Start the laptop database with `docker compose -f deploy/compose.dev.yaml up -d` and run `python manage.py migrate`.

*Done means:* the API docs page loads, the database is reachable, and the envelope works for one test success and one test error.

### Step 2 (week 1): Models and data

- [ ] **B2.1** Models and migrations for all apps: `accounts` (User, Student, OTP, SignupAttempt), `elections` (Election, Position, Candidate), `voting` (ElectionToken, VoteRecord, Ballot), `content` (Post, Administration, Executive), `audit` (AuditLog).
- [ ] **B2.2** Register models in the Django admin. `Ballot` and `AuditLog` are **read-only**.
- [ ] **B2.3** Check the Ballot table: no foreign key to Student or User, no timestamp, UUID ids.
- [ ] **B2.4** Write `import_students <csv> [--dry-run]` with the same checks as the API import (consistent matric format, valid and unique lowercase emails, no blank fields, level is a number). It prints a report and never overwrites an activated student's email without `--force` and a reason.
- [ ] **B2.5** Write `seed_fake_data --students 12000`. It refuses to run when `DEBUG=False`.
- [ ] **B2.6** Write `create_admin` (first admin, prints the authenticator setup link once).
- [ ] **B2.7** Deploy to staging once staging exists (week 2) and load fake data there.

*Done means:* 12,000 fake students load, the import dry run produces a sensible report, and test group 8 (secrecy) passes.

### Step 3 (week 2): Authentication

- [ ] **B3.1** CSRF endpoint.
- [ ] **B3.2** Sign-up in three steps: `start` (email plus matric match, OTP sent only on a match), `verify` (OTP), `complete` (set password). Sign-up does not log the student in.
- [ ] **B3.3** Student login (matric number plus password) and staff login (email, password and authenticator code). Logout. `/auth/me`.
- [ ] **B3.4** Password reset in three steps (forgot, verify, reset). Stays open during voting.
- [ ] **B3.5** `send_otp_email` and the plain-text and HTML templates. The text must say: the student's name, the code, that it expires in 5 minutes, that **AUSA** will never ask for it, and to ignore the email if they did not request it. No links.
- [ ] **B3.6** OTP rules: 6 digits, valid 5 minutes, one use, stored as a hash, resend after 60 seconds, 5 wrong tries then a 15 minute lock, 5 emails per student per day, global hourly cap, `503 EMAIL_CAPACITY_REACHED` with `retry_after_seconds` when a cap is hit.
- [ ] **B3.7** Mismatch handling: one neutral message, a `SignupAttempt` record, 5 mismatches per IP per hour then `SIGNUP_LOCKED` for 1 hour. Never touch the real student's record.
- [ ] **B3.8** Throttles and django-axes (lock on matric number plus IP, 5 failures, 15 minutes). Behind Caddy, set the proxy settings so limits see the real client IP.
- [ ] **B3.9** Bounced email handling: set `email_status = bounced` and stop sending.
- [ ] **B3.10** Audit log helper `log_event()` and the first audit entries.

*Done means:* tests 1, 2, 3, 4 (reset part), 12 and 13 pass; every `/auth` endpoint matches the contract on the docs page and is deployed to staging. Tell the frontend lead.

### Step 4 (week 3): Elections admin

- [ ] **B4.1** Elections: create, list, edit (until start), delete (only while draft).
- [ ] **B4.2** Schedule with `token_window_opens_at`, `start_at`, `end_at`. Setting the schedule stores `original_end_at`.
- [ ] **B4.3** Positions with scope (all, faculty, department) and candidates. Photos: JPG or PNG, max 2 MB, checked with Pillow, renamed to random names, stored outside the code folder. Manifestos and names are plain text.
- [ ] **B4.4** Extend (admin only, while live, only later, reason required) and reopen (admin only, after the end and before publishing, future end time, reason required). Both audit-logged with old time, new time and reason.
- [ ] **B4.5** Sign-up lock: sign-up refused with `ACTIVATION_CLOSED` from `start_at` to the current `end_at`, including extensions and reopenings, using the server clock.
- [ ] **B4.6** Sign-up attempts list for officers.
- [ ] **B4.7** Role permissions on every view. A student gets 403 on every `/admin` URL; staff without a verified authenticator session get 403.

*Done means:* tests 4, 10 (extend and reopen part) and 11 pass; officer and admin election endpoints are on staging.

### Step 5 (week 4): Tokens

- [ ] **B5.1** `token_status` in `/auth/me` (`not_open`, `available`, `shown`, `used`, `closed`).
- [ ] **B5.2** `token/reveal`: returns the token once, then `TOKEN_ALREADY_SHOWN`. 12 characters from the safe alphabet, shown as `XXXX-XXXX-XXXX`, stored only as an HMAC hash. Response has `Cache-Control: no-store` and is never logged.
- [ ] **B5.3** Self reissue with a fresh OTP until `start_at`; the old token is revoked.
- [ ] **B5.4** Help desk reissue by an officer: needs `id_card_checked` and a reason, refused if the student has voted, only when `HELPDESK_REISSUE_DURING_LIVE=True`. Audit-logged.
- [ ] **B5.5** Admin token-status counts (never token values).

*Done means:* test 5 passes; the raw token appears nowhere in the database or logs.

### Step 6 (week 4 to 5): Voting and results

- [ ] **B6.1** `token/verify`: 5 wrong tries then `TOKEN_LOCKED` for 15 minutes; opens a 20 minute ballot session.
- [ ] **B6.2** Ballot: only the posts this student may vote for, candidates shuffled per student.
- [ ] **B6.3** Vote submit in one database transaction: check token, create VoteRecord, create Ballot rows (shuffled insert order), mark token used. Checked against the current end time. The unique rule on (election, student) is the last lock.
- [ ] **B6.4** Dashboard states in `/auth/me` and `/elections/current` (`voting_status`, `voted_hour`, `receipt_id`, `server_time`, extension fields).
- [ ] **B6.5** Turnout (voted over eligible, faculty breakdown, never candidate counts).
- [ ] **B6.6** Officer results (only after the current end time, every view logged), publish (admin only, only after the end, cannot be undone), student results.

*Done means:* tests 6, 7, 8, 9 and 11 pass, including the two-requests-at-once test, which must produce exactly one vote.

### Step 7 (week 5): Content (cut first if short on time)

- [ ] **B7.1** Posts (body cleaned with nh3), administrations and executives with photos.
- [ ] **B7.2** Public read endpoints and officer management endpoints.

### Step 8 (weeks 6 to 7): Finish

- [ ] **B8.1** Fix pen test round 1 findings (week 6).
- [ ] **B8.2** Backups, logging and production settings.
- [ ] **B8.3** Full test suite green. `pip-audit` clean. `python manage.py check --deploy` clean with production settings.
- [ ] **B8.4** README: setup on Windows (venv, Docker Desktop for the database, why not XAMPP), running tests, environment variables, deploying with Docker Compose, setting the public address for Option A or B, how to switch the email provider, and the list of pending CONFIRM items.
- [ ] **B8.5** Load test fixes and second pen test fixes. **Code freeze at the end of week 7.**

### The 13 test groups that must pass

1. Sign-up match and mismatch rules
2. OTP rules
3. Set password
4. Activation lock
5. Token reveal and reissue
6. Token verify
7. Vote, including the race test
8. Ballot secrecy
9. Results rules
10. Extend and reopen
11. Permissions
12. Error envelope
13. Email switching and bounces

---

## Section 5. Phase 2: Frontend build (weeks 1 to 7) [F]

The frontend lead builds every screen against **mock responses** copied from the contract from week 1, then switches each area to the real API when the backend says it is on staging.

### Rules that must not slip

- No tokens or secrets in localStorage. No election token in the address bar, logs or analytics.
- Never `dangerouslySetInnerHTML` without DOMPurify.
- Countdowns use `server_time` from the API, never the phone clock.
- The vote button is disabled after the first click.
- The token is cleared from memory when its pop-up closes.
- Send cookies with every API call and the `X-CSRFToken` header on every POST, PATCH and DELETE.
- Poll `/elections/current` every 60 seconds so an extension or reopening shows up ("Voting extended to 7:00 PM").
- Do not cache token responses in the data-fetching library.

### Week 1: Setup (no real API yet)

- [ ] **F1.1** React, Vite and TypeScript project. Routing.
- [ ] **F1.2** API client: cookies on, CSRF header, error envelope handling.
- [ ] **F1.3** Mock server (Mock Service Worker) with responses copied from the contract.
- [ ] **F1.4** Components from Figma: Button, Input, Read-only input, Badge, Logo, page shells.

### Week 2: Sign-up and login (switch to real `/auth` at the end of the week)

- [ ] **F2.1** Sign-up page: email and matric number. Show the neutral "We couldn't verify these details" message on `DETAILS_NOT_MATCHED`.
- [ ] **F2.2** Enter-the-code page with a 60 second Resend countdown.
- [ ] **F2.3** Set-password page: matric number **read-only**, password and confirm password.
- [ ] **F2.4** Welcome page ("Thank you for coming on board!") then the login page.
- [ ] **F2.5** Student login, forgot password flow. (Staff login screen with the authenticator code field.)

### Week 3: Officer and admin console (switch to election admin endpoints)

- [ ] **F3.1** Elections list and create, schedule form with the token window.
- [ ] **F3.2** Positions and candidates with photo upload.
- [ ] **F3.3** Extend and reopen dialogs (reason required).
- [ ] **F3.4** Sign-up attempts list.

### Week 4: Student dashboard and voting (switch to tokens and voting)

- [ ] **F4.1** Dashboard states: not open, ready, voted, closed, results.
- [ ] **F4.2** **Token pop-up:** cannot be closed by clicking outside or pressing Escape; large spaced monospace token; Copy and Download buttons; warning text; "I have saved my token" checkbox enables Close.
- [ ] **F4.3** Token entry page, ballot, review page, confirmation pop-up ("this cannot be changed").
- [ ] **F4.4** After voting: back to the dashboard showing Voted, the hour and the receipt number; Vote now disabled.
- [ ] **F4.5** 60 second polling for extensions and reopenings.

### Week 5: Results and public pages (switch everything)

- [ ] **F5.1** Results on the dashboard, admin publish screen.
- [ ] **F5.2** Voter roll import screen, audit log screen.
- [ ] **F5.3** Public pages: home, blog, executives, privacy notice (blog and executives are cut first if short on time).

### Weeks 6 to 7: Polish and freeze

- [ ] **F6.1** Fix pen test findings.
- [ ] **F6.2** Phone and tablet layout, slow network states, accessibility (labels, focus, contrast, keyboard).
- [ ] **F7.1** Production build and final copy.
- [ ] **F7.2** Check on cheap Android phones and slow data (every screen usable on 3G). **Freeze at the end of week 7.**

---

## Section 6. Phase 3: Infrastructure and deployment [B]

One address for everything: Caddy sends `/api/*` to Django and everything else to the React app, so there is no CORS setup, one certificate and one DNS record.

### Staging (week 2)

- [ ] **I1** Rent a small server (about 1 vCPU, 2 GB RAM). Ubuntu 24.04, normal user, firewall open on 80 and 443, SSH from your own IP only, key login only.
- [ ] **I2** Install Docker Engine and the Compose plugin. Clone the backend repository. Write `deploy/.env` (never committed).
- [ ] **I3** Point the staging address at the server (Option A: Cloudflare DNS. Option B: ask ICT for a staging record, or stage on a cheap domain or a Cloudflare Tunnel; only production needs the school name).
- [ ] **I4** Copy the React build into `deploy/frontend-dist`, run `docker compose up -d --build`, then run migrations.
- [ ] **I5** GitHub Actions: tests must pass before anything deploys to staging.
- [ ] **I6** Fake data only. Real student data never goes on staging.

*Done means:* the staging address opens over HTTPS and `/api/docs/` loads.

### Production (week 7)

- [ ] **I7** Same steps on a bigger server (about 2 vCPU and 4 GB RAM) with the chosen address.
- [ ] **I8** Cloudflare in front for Option A (or Option B if the school's DNS is on Cloudflare).
- [ ] **I9** Switch email to the school Workspace account in `.env`. Send test codes to school inboxes and confirm they arrive within a minute.
- [ ] **I10** Daily database dump by cron, copied **off** the server, and one real restore tested.
- [ ] **I11** Error alerts (Sentry free tier) and an uptime check that messages both leads.
- [ ] **I12** Admin and officer accounts with strong passwords and authenticator apps.
- [ ] **I13** A few days before election day, resize to about 4 vCPU and 8 GB and run the load test on it. Resize back after results are published.
- [ ] **I14** Before sign-up opens: `pip-audit` and `npm audit` clean, `manage.py check --deploy` clean, API docs turned off or staff-only, address fixed.

---

## Section 7. Phase 4: Testing and security [ALL]

| When | Test | Who | Pass when |
|---|---|---|---|
| Every pull request | Backend tests (pytest), frontend tests (vitest) | Both leads | All green before merge |
| Weeks 2 to 4 | Rule tests: email and matric match, OTP limits, sign-up lock, token shown once, one vote, race test, results locked | Backend lead | Every test in the agent prompt passes |
| Week 4 | Pen testers' written test plan is ready | Pen testers | Plan reviewed by both leads |
| Week 5 | **Pen test round 1** on staging | Pen testers | Written findings, each fixed and retested |
| Week 6 | **Mock election** with 30 to 50 volunteers, including some who "lose" their token and one admin extend | Whole team | Counts match the expected result exactly; problems logged |
| Week 7 | **Load test:** hundreds of logins and votes within minutes | Backend lead | No errors; pages load in under 2 seconds on staging-sized hardware |
| Week 7 | **Pen test round 2** on the production setup | Pen testers | No high or critical findings left |
| Week 8 | Final phone check on low-end Android and slow data | Frontend lead | Every screen usable on 3G |

### Pen test tasks

- [ ] **P1 [P]** Read Blueprint v4 and write the test plan (by week 4).
- [ ] **P2 [P]** Account takeover through sign-up and password reset, OTP guessing and bypass, signing up with someone else's matric number, skipping the OTP step.
- [ ] **P3 [P]** Token tests: seeing a token twice, reissuing someone else's token, using a revoked token, guessing tokens, abusing the help desk endpoint.
- [ ] **P4 [P]** Signing up while an election is live.
- [ ] **P5 [P]** Viewing or publishing results while live, extending or reopening without admin rights or after publishing.
- [ ] **P6 [P]** Voting twice, voting without a token, voting outside the time window, two requests at the same instant.
- [ ] **P7 [P]** Linking a vote to a student (including by ID order or timing).
- [ ] **P8 [P]** XSS in blog, manifesto, names and image uploads. CSRF. Broken access control between roles.
- [ ] **P9 [P]** Email bombing through the OTP endpoint, and locking real students out with wrong logins.
- [ ] **P10 [ALL]** Fix every finding, retest, and record the result.

---

## Section 8. Phase 5: Launch, real data and the sign-up window (weeks 8 to 10)

- [ ] **L1 [SCHOOL][B]** Real data in (week 8, day 1 to 2). The school sends the CSV through the agreed secure channel. Run the import as a **dry run**, send the error report back, fix, then import for real. Get written sign-off on the row count.
- [ ] **L2 [B]** Create the election. Officers add positions and candidates and set the token window (opens with sign-up), start and end times.
- [ ] **L3 [ALL]** Announce through class reps, WhatsApp groups, notice boards and the AUSA page: what to do, the sign-up deadline (the election start time), and the help desk location.
- [ ] **L4 [B]** Open sign-up (start of week 8). Check every day: the email count against the daily limit, and the unmatched sign-up attempts.
- [ ] **L5 [SCHOOL]** Run the help desk (weeks 8 to 10). Fix students whose email or matric is wrong in the school list (admin only, with a reason) and help anyone stuck.
- [ ] **L6 [ALL]** Freeze (election day minus 1): final backup, check the schedule, confirm admin accounts and who may extend or reopen.

*Done means:* nearly all eligible students have activated, and no emails were lost to the daily cap.

---

## Section 9. Phase 6: Election day runbook

| Time | What happens | Who |
|---|---|---|
| 1 hour before | Backup taken. Check the server, email and uptime alerts. Admin logs in with the authenticator app. | Backend lead |
| Start time | Voting opens automatically; sign-up closes automatically. Check the first few votes land and turnout moves. | Backend lead |
| During voting | Watch errors, server load and turnout. Help desk handles lost tokens with an ID check. | Both leads, help desk |
| If something breaks | Fix it, then the admin extends the closing time with a written reason. Dashboards update within a minute. | Admin, with committee approval |
| End time | Voting closes. Take a backup immediately. | Backend lead |
| After closing | Officers review counts. Admin publishes results; they appear on every student's dashboard. If voting must continue, reopen **before** publishing. | Electoral committee, admin |

---

## Section 10. Phase 7: After the election and handover (week 11)

- [ ] **H1** Final backup kept somewhere safe and off the server.
- [ ] **H2** Audit log exported for the electoral committee.
- [ ] **H3** Student data deleted or kept exactly as the data agreement says. Copies on laptops deleted.
- [ ] **H4** Short report to the school: turnout, problems, fixes, what to change next time.
- [ ] **H5** Handover: admin accounts, server access, code, a run guide for the next election, and who maintains it.
- [ ] **H6** Resize the production server back down after results are published.
- [ ] **H7** Agree maintenance (who looks after the system after handover) in writing.

---

## Section 11. Risks and what we do

| Risk | What happens | What we do |
|---|---|---|
| Student list arrives late or messy | Sign-up window shrinks, election slips | Ask for the sample CSV in week 0, dry-run the import, agree the delivery date in writing |
| School email account delayed or app password refused | No OTPs, nobody can sign up | Ask ICT in week 0; Amazon SES ready as a fallback |
| 2,000 emails a day is not enough at the rush | Students wait for codes | Open sign-up 14 days early; queue emails; open by faculty if needed |
| Students lose tokens | Help desk queues on election day | Clear pop-up warning, reissue before voting, staffed help desk |
| Backend falls behind | Frontend stuck on mocks | Weekly check-in on the contract order; cut blog and executives pages first |
| Server problem during voting | Some students cannot vote | Uptime alerts, backups, admin extend or reopen |
| Election date is too close | Not enough time to test | Fix the date in week 0, or cut scope using 1.7 |

---

## Section 12. Open decisions (CONFIRM) with their deadlines and defaults

| Decision | Default until decided | Needed by |
|---|---|---|
| Election date | None. Must be set. | Week 0 |
| Any posts limited to a faculty or department? | Built as a setting (scope on each post) | Start of week 8 |
| May students abstain? | `ALLOW_ABSTAIN=True` | Start of week 8 |
| May the help desk reissue a lost token during voting? | `HELPDESK_REISSUE_DURING_LIVE=True` | One week before election day |
| Who may extend or reopen: admin alone, or with the electoral committee's written approval? | Admin only | One week before election day |
| Results as totals only, or with a faculty breakdown? | Totals | Week 5 (the screens are built then) |
| How long is student data kept, and who owns the system after handover? | None | With the data agreement (week 8) |
| Who pays for domain, hosting and email, and who maintains the server after the election? | None | Before handover (week 11) |
| Voting from anywhere, or campus networks only? | Anywhere | Week 5 |
| Option A or Option B for the public address | None | Start of week 2 |
| Delete sign-up attempt records after the election? | Delete | With the data agreement |

---

## Section 13. Weekly rhythm

- **Monday:** both leads check this file, tick finished tasks, and agree the week's tasks. Check the three date-deciding items (election date, school email account, real student list).
- **During the week:** backend finishes a module, deploys it to staging, and tells the frontend lead. Frontend switches that area from mocks to the real API.
- **Friday:** run the tests, run `pip-audit` and `npm audit`, and update the Blocker log below.

## Section 14. Blocker log

| Date | Task | What is blocking | Who can unblock | Status |
|---|---|---|---|---|
| | | | | |

## Section 15. Decision log

| Date | Decision | Decided by | Documents to update |
|---|---|---|---|
| | | | |
