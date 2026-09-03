# PulseID

**A universal digital medical ID.** One record per person, built around their
National ID — instantly readable by any doctor with a login, and instantly
readable by a first responder with just a QR scan, with nothing more than
life-critical facts ever exposed to a stranger. It also installs like a real
app on your phone or laptop — no app store required.

This repo is split into three independently deployable services, the way a
real product ships:

```
pulseid/
├── backend/     Express + TypeScript API — auth, records, appointments, QR, audit log, risk scoring, proactive follow-ups, analytics aggregates
├── frontend/    Next.js app (installable PWA) — doctor portal, patient portal, emergency page
└── analytics/   Next.js app — national/regional public-health dashboard (aggregate counts only)
```

## Why this split

- **`backend/`** owns the only copy of the SQLite database, all business
  logic, session signing, rate limiting and the audit trail. It's a plain
  Express API — deploy it anywhere Node runs (Render, Fly.io, a VPS, a
  Docker container next to a Postgres box once you outgrow SQLite).
- **`frontend/`** is a pure Next.js app. It never touches a database — every
  page either fetches from the backend at request time (for fast,
  server-rendered pages) or from the browser (for interactive forms), and
  ships zero server secrets other than the shared session-verification key.
- **`analytics/`** is a separate Next.js app for a completely different
  audience — a public-health analyst or Ministry-of-Health user, not a
  doctor or patient. It has its own login and its own session secret, and
  it never has a path to an individual patient record: it only calls
  `backend`'s `/api/analytics/*` routes, which return region/time-bucketed
  counts (with small-cell suppression — see its own README) and nothing
  with a name or National ID attached.

`frontend` trusts the same `SESSION_SECRET` as `backend` to verify JWT
session cookies, which is what lets Next.js middleware protect `/doctor/*`
and `/patient/*` routes without an extra network round-trip on every click,
while the backend remains the single source of truth that actually issues
those cookies. `analytics` is deliberately **not** part of that trust
circle — it authenticates to `backend` with its own service-to-service key
(`ANALYTICS_SERVICE_KEY`) and has its own independent analyst session, so a
clinical credential and an analytics credential can never be swapped for
each other, on purpose.

## What's inside

### Doctor portal
Email/password login, search patients by name or National ID, scan a
patient's QR to jump straight to their file, register new patients, add
visits and prescriptions, and edit a patient's core details after the fact
(see below). Patients are shown in a sortable directory table — click any
column header to sort by name, age, or registration date, with a
mobile-friendly card view on small screens.

### Hospital admin console
A separate email/password login at `/hospital-admin` for hospital staff
who manage the hospital itself rather than treat patients: an overview
dashboard of the hospital's own patient/appointment/doctor activity,
add/deactivate doctor accounts (one at a time or via bulk CSV import),
view a doctor's activity log read-only, and fix patient registration
details (name, contact info, blood group, etc.) without ever seeing
medical history — see
[Hospital admin console](#hospital-admin-console) below for the full scope.

### Patient portal
Only doctors and hospital staff can register a patient; patients can never
create their own PulseID. Patients sign in by scanning the QR code printed
on their CNIC (or typing it in manually), then verifying a 6-digit code sent
to the phone number on file (demo mode returns the code on-screen since
there's no SMS gateway wired up). Once verified, patients land on **My
Reports**, with a full visit timeline, downloadable PDF/JSON report, and a
complete audit log of every time someone has opened their file.

The header nav (Home, Records, Appointments, My Reports, Dependents,
Emergency Scan, Emergency Access, Audit log) collapses to a hamburger menu
below `xl` — eight items is too many to ever fit in one row on a phone or
tablet, so rather than shrinking or wrapping them into a mess, the full row
only appears once there's genuinely enough width for it.

**Emergency Scan** lets a patient preview exactly what a doctor or first
responder sees when they scan a CNIC/B-Form or a PulseID QR — the same
scoped, life-critical-only view used everywhere else. It's reachable two
ways:
- Signed in, from `/patient/emergency-scan` (in the header nav, and as a
  quick-access button on **My Reports** and **Emergency Access**).
- Signed out, from `/emergency/scan` — a deliberately public route, since a
  first responder or bystander scanning someone else's card will never have
  a PulseID login of their own. It's linked directly from the patient login
  screen.

### Appointments — request vs. schedule are two different roles
Patients can **request** an appointment (pick a doctor, optionally say why),
but they can never choose the date or time — that's deliberately reserved
for the doctor/clinic:

- **Patient side:** "Request an appointment" only asks for a doctor and an
  optional reason. The request goes in as `status: requested` with no date
  attached.
- **Doctor side:** a requested appointment shows "Choose time & confirm" —
  the doctor picks the date/time and confirms in one step. The API rejects
  any attempt to confirm an appointment before a time has been set. Doctors
  can also **reschedule** a confirmed appointment, mark it completed, or
  cancel it.
- **Patients** can cancel their own request or confirmed appointment, but
  there is no route — client or server — that lets a patient set or change
  the date/time. This is enforced on the backend, not just hidden in the UI.

**Reminders.** A standalone process (`backend/scripts/reminder-scheduler.ts`,
`npm run reminders`) sweeps every 15 minutes for confirmed appointments
landing in roughly the next 20–28 hours and haven't been reminded about
yet, and emails and/or texts the patient once — `reminder_sent_at` on the
appointment prevents a duplicate even if a run is delayed. It shares
`backend/src/lib/reminders.ts` with any future manual "send now" trigger,
so a scheduled reminder and a manual one go through the same code path.
Email goes through SMTP (`backend/src/lib/email.ts`, `SMTP_*` env vars);
SMS reuses the same Twilio wiring as OTP delivery. Configure either, both,
or neither — with neither configured the scheduler just logs what it would
have sent, so the reminder flow is still visible in dev without real
credentials. Runs as its own container (`backend/Dockerfile.reminders`,
the `backend-reminders` service in `docker-compose.yml`) sharing the same
SQLite volume as the main API, mirroring how `analytics-scheduler` is set
up for bulletins.

**Recurring appointments.** When confirming a request, a doctor can tick
"Repeat for follow-ups" and choose a cadence (weekly/every 2 weeks/monthly)
and a count (2–26 occurrences) — useful for chronic-condition follow-ups
that would otherwise mean re-requesting and re-confirming the same visit
every time. This creates the requested number of already-confirmed
appointments, spaced out from the first one's date/time, all sharing a
`recurrence_group_id` so the UI can show "3 of 6" on each occurrence. Every
occurrence is independently reschedulable/cancellable afterwards — this
only sets up the initial series, it deliberately doesn't keep them linked
for cascading edits.

**Waitlist.** An alternative to requesting a specific slot from a fully
booked doctor: a patient can join that doctor's waitlist instead (with an
optional reason), and the doctor's queue shows waiting patients oldest
first. One action — "Offer a slot" — turns a waitlist entry directly into
a confirmed appointment at a doctor-chosen time, instead of the patient
separately re-requesting and the doctor separately confirming.

**Calendar/agenda view.** `/doctor/calendar` gives doctors a week-by-week
grid of everything with a set time — a different shape of the same data as
the flat, status-filtered list at `/doctor/appointments`, better suited to
"what does my Tuesday look like" than a status queue is. Backed by
`GET /api/doctor/calendar?start=&end=`, a date-range query separate from
the existing status-filtered endpoint.

### Risk scoring — a rule-based, explainable flag from vitals

When a doctor adds a visit, the vitals fields (blood pressure, blood sugar,
body temperature, heart rate) are optional. If at least one is filled in,
`backend/src/lib/risk-scoring.ts` computes a `low`/`mid`/`high` flag from a
fixed set of clinical thresholds — hypertensive/hypotensive blood pressure
bands, fasting-equivalent glucose bands, fever/hypothermia cutoffs,
tachycardia/bradycardia cutoffs, plus an age factor — with an optional
`maternal` context that shifts several of those thresholds toward
pregnancy-related risk (pre-eclampsia-range blood pressure, extremes of
maternal age).

This is deliberately **not** a trained ML model. A transparent, auditable
threshold check is a better fit for a medical record than an opaque
classifier: every score comes back with a `factors` array explaining
exactly which readings triggered it, in plain language, and every response
carries a fixed disclaimer (`RISK_DISCLAIMER`) — *"Automated, rule-based
flag from recorded vitals only — not a diagnosis. Always use clinical
judgement."* — so that context can never get lost between the API and
whatever's rendering it.

The doctor sees the flag immediately after saving a visit with vitals
attached, plus a running risk badge and factor list on the patient's own
detail page (`GET /api/patients/:id/risk`). Patients can see the same
score for their own record via `GET /api/patient/risk` — it's their own
health data, computed the same way a doctor sees it, not a separate
"softened" version.

### Proactive follow-ups — scheduled check-ins for high-risk patients

A doctor can start a recurring follow-up for one patient — postpartum,
post-operative, chronic-condition monitoring, or a custom type — choosing
only a follow-up type and how often to check in (`POST
/api/doctor/followups`). `backend/src/lib/followup-agent.ts` ships sensible
default question sets per pathology (a mix of 0–10 scale, yes/no, and
free-text questions) so the doctor doesn't have to write them by hand,
though custom questions can be supplied too.

A standalone scheduler (`backend/scripts/followup-scheduler.ts`, `npm run
followups`) sweeps hourly for agents whose next check-in is due, prompts
the patient by email/SMS (reusing the same `email.ts`/`sms.ts` wiring as
appointment reminders, with the same demo-mode fallback if neither is
configured), and advances the schedule. The patient answers from
`/patient/followups` (a banner surfaces anything pending right on their
home page).

Risk flagging on a submitted check-in is **deterministic**, not AI-based:
each scale question carries a `concernAt` threshold set by the default
templates, and an answer at or above it flags the check-in `concern` (or
`urgent` if well above it) — this keeps alerting reliable with zero
external dependencies. An AI-written 2–4 sentence doctor-facing summary
(`FOLLOWUP_SYSTEM_PROMPT` in `lib/ai.ts`, a different trust boundary from
the hospital-admin briefing prompt since it's given one patient's own
answers) is layered on top when an AI provider is configured, but it's a
convenience, never the thing alerting depends on. Every unacknowledged
`concern`/`urgent` check-in surfaces on the doctor's `/doctor/followups`
dashboard until they acknowledge it.

Runs as its own container (`backend/Dockerfile.followups`, the
`backend-followups` service in `docker-compose.yml`) sharing the same
SQLite volume as the main API — mirroring how `backend-reminders` is set
up.

### Doctor-only patient editing
Doctors can open **✎ Edit details** on a patient's file to correct or update
their phone number, address, blood group, allergies, chronic conditions,
weight, and pediatrician info. National ID and ID type are intentionally
excluded — that's the patient's fixed legal identity, not something to
correct from a form. The `PATCH /api/patients/:id` route this calls is
gated by `requireDoctor`; there is no equivalent route reachable from a
patient session.

### Hospital admin console
A third staff-facing frontend, at `/hospital-admin`, separate from the
doctor and patient apps (its own login, its own cookie, its own layout) —
see [Regional hierarchy](#regional-hierarchy-region--hospital--doctor)
above for the full permission model. Five things live here:

- **Overview** (`/hospital-admin/overview`) — patient volume, appointment
  load (by status), and per-doctor activity (visits recorded, appointments,
  last active), scoped to the admin's own hospital. This is the same shape
  of aggregate the analytics service already computes per-hospital for
  analysts, just surfaced directly to the hospital that owns the data —
  and unlike the analytics side, these counts are never small-cell
  suppressed, since an admin viewing their own hospital's real numbers
  isn't the cross-hospital re-identification risk that suppression exists
  to prevent.
- **Doctor accounts** (`/hospital-admin/dashboard`) — add a doctor one at a
  time, edit their specialization inline, deactivate/reactivate.
  `hospital_id` always comes from the admin's own session server-side, so
  there's no request shape that creates or edits a doctor at a different
  hospital. A doctor's **recent activity** is viewable read-only from here
  too — what they've done, for which patient, and when, across their last
  100 actions — without exposing their login or any clinical content
  (diagnoses/notes never appear in this view, only the action/patient/time).
- **Bulk doctor import** — a hospital onboarding 20+ doctors at once can
  upload a CSV (`fullName,email,password,licenseNumber,specialization`)
  instead of creating accounts one by one. Each row is validated and
  created independently — a typo in one row is reported and skipped, it
  doesn't block the rest of the file — and the response lists a per-row
  outcome so the admin knows exactly what to fix and re-upload.
- **Patient registration fixes** (`/hospital-admin/patients`) — search by
  National ID or name, then correct the same demographic fields a doctor
  can (name, DOB, contact info, blood group, allergies/conditions *on
  file*, pediatrician info). This calls `PATCH /api/hospital-admin/patients/:id`,
  which runs the identical validation as the doctor route, and every edit
  is audit-logged with `actor_role: "hospital_admin"` so it's
  distinguishable from a doctor's edit in the patient's own audit trail.
- **Appointments** (`/hospital-admin/appointments`) — every appointment
  across every doctor at the hospital in one filterable log (status, doctor,
  date range, free-text search over patient/doctor/reason), with CSV export
  of whatever's currently filtered. An optional "Summarize load" button asks
  an LLM to draft a short operational briefing — bottlenecks, an overloaded
  doctor, requests that have been waiting too long — built only from
  aggregate counts by status/doctor (`getHospitalAppointmentLoadSummary`);
  the prompt never includes a patient name, reason, or any clinical detail.
  Requires `OPENROUTER_API_KEY` or `ANTHROPIC_API_KEY` in the backend's env;
  without one, the button shows a plain "not configured" message instead of
  failing.

What it deliberately **doesn't** do: this console never renders visit
history, diagnoses, symptoms, notes, or prescriptions, and there is no
route under `/api/hospital-admin/*` that returns any of them — clinical
charting stays exclusively behind `requireDoctor`. The doctor activity
view above is deliberately read-only and summary-level for the same
reason: it tells an admin *that* a doctor opened a record, never *what's
in it*. The distinction is "who's allowed to fix a typo in someone's phone
number, or see that a doctor is keeping up with their caseload" (an
administrative/front-desk concern) versus "who's allowed to read or write
what a doctor diagnosed" (a clinical concern) — this app draws that line
at the API layer, not just by hiding UI, so it holds even against a
direct request to the backend.

### Emergency access
Scanning a patient's CNIC (or their PulseID QR, if a hospital issued one)
opens a public page with *only* blood group, allergies, chronic conditions
and emergency contacts — no diagnoses, no visit history. The QR token
rotates on every scan, so a photographed or leaked code can never be
replayed.

### Detailed medical reports
One report engine, two audiences. `/patient/report` gives patients a
plain-language, printable copy of their whole record. `/doctor/patients/:id/report`
gives clinicians an official letterhead version of the same data — visit
history, current medications, prescription history, risk/allergy alerts,
emergency contacts, and a record-access audit trail — ready to print or save
as a PDF.

### Installable as a web app (PWA)
PulseID installs like a native app — "Install app" in Chrome/Edge, or "Add
to Home Screen" on iOS/Android — with its own icon, no browser chrome, and a
minimal offline fallback screen if the connection drops mid-use. A service
worker (`frontend/public/sw.js`) caches only the static app shell; it
deliberately never caches API responses or medical data, so installed users
always see fresh, live data when online and a clear "you're offline" screen
when they're not — never stale patient information passed off as current.

The landing page (`frontend/components/LandingApp.tsx`) has two views: an
**app-style launcher** (four big portal buttons — Patient / Doctor /
Hospital Admin / Emergency Scan) and the **full marketing website**. A pill
button top-left ("About PulseID" / "Back to app") toggles between them —
available whether you're in a plain browser tab or the installed app — and
the choice is remembered in `localStorage`. Installed launches default to
the app launcher; browser tabs default to the website.

### Analytics — national/regional dashboard, aggregate-only
A third, separate app for a different kind of user entirely: a public-health
analyst who needs to know *"which region has how many cases of what"*, not
*"what is patient X's medical history."* It has its own password-gated
login (`analytics/`), its own session cookie, and it only ever talks to
`backend`'s `/api/analytics/*` routes over a shared service key — there is
no route it can call, and no route the backend exposes to it, that returns
a patient's name, National ID, or individual record.

What it shows:
- **Overview** — national totals (visits, patients, hospitals, active
  regions) plus the busiest regions and most common diagnoses nationwide.
- **Regions** — one row per hospital city (visit volume, distinct patients
  seen, top condition), each with a hospital-level drill-down underneath
  (every hospital in that region, its doctor count, and its own visit/patient
  counts) — see "Regional hierarchy" below for how this rolls up.
- **Map** — a Leaflet map, circles sized/colored by regional case volume.
- **Trends** — pick a diagnosis (and optionally a region), see case counts
  over time with a toggleable forecast overlay (a simple linear
  projection) — the view built for spotting an outbreak forming before
  it's obvious in raw numbers.
- **Benchmark** — one region's share of visits per condition vs that
  condition's share nationwide, to see what a region sees disproportionately more of.
- **Alerts** — regions/conditions running well above their own recent
  baseline, flagged by a simple, explainable ratio — not a black-box model.
- **Ask** — a natural-language question box, answered by an LLM that only
  ever sees the same aggregate JSON every other tab already shows.
- **Bulletins** — the flagship feature: a scheduled job
  (`analytics/scripts/scheduler.ts`) generates a weekly/monthly
  epidemiological bulletin — AI-drafted narrative plus four real charts
  (national trend, busiest regions, top conditions, alert magnitude) —
  and it sits in a review queue as `pending_review` until an analyst
  explicitly approves or rejects it. Nothing auto-publishes. A manual
  "Generate now" button on the page uses the identical code path for
  on-demand bulletins.
- **Resources** — AI-drafted "worth a closer look" suggestions, cross-
  referencing visit load, alerts, and reporting gaps — framed as pointers
  for a human planner, never as decisions.
- **Data quality** — flags regions whose reporting volume dropped or spiked
  sharply, since that's usually a broken pipeline, not real disease change.

The four AI-powered tabs (Ask, Alerts, Bulletins, Resources) go through
`analytics/lib/ai.ts`, which supports **OpenRouter** (`OPENROUTER_API_KEY`,
used automatically if set — lets you pick from many underlying models
through one key) or **Anthropic direct** (`ANTHROPIC_API_KEY`) as a
fallback. Either way, every AI feature is a pure *writer over
already-computed aggregate data* — the model never generates a number
itself (every percentage/count in a bulletin was computed in
`backend/src/lib/repo.ts` first) — and each degrades to a plain "not
configured" message if neither key is set, rather than breaking the page.

Bulletins are stored in analytics' own small SQLite database
(`analytics/lib/bulletin-db.ts`), completely separate from the backend's
patient-record database — it only ever holds generated narrative text, a
chart-data snapshot, and a review status.

Every count under 5 is shown as `<5` instead of an exact figure
(`MIN_CELL_SIZE` in `backend/src/lib/repo.ts`) — standard small-cell
suppression, the same principle public-health agencies use, so a rare
diagnosis in a small region can never be combined with location to
re-identify one person. See `analytics/README.md` for the full trust-model
writeup and feature-by-feature detail.

## Regional hierarchy: region → hospital → doctor

Patient data organizes around a three-level hierarchy: a **region** (one of
Pakistan's provinces, plus Gilgit-Baltistan, Azad Jammu & Kashmir, and the
Islamabad Capital Territory) contains **hospitals**, and each hospital
contains **doctors**. This has always existed at the database level
(`hospitals.province`/`hospitals.city`, `doctors.hospital_id`) — what this
section documents is the login/permission layer built on top of it, and how
analytics rolls counts up through all three levels rather than skipping
straight from region to doctor.

```
Gilgit-Baltistan
├── DHQ Gilgit                          (hospital-admin login)
│   ├── Dr. Amina Baig — Child Specialist       (doctor login)
│   └── Dr. Karim Hunzai — General Medicine     (doctor login)
└── Skardu Civil Hospital               (hospital-admin login)
    ├── Dr. Fatima Sheikh — General Medicine    (doctor login)
    └── Dr. Zubair Baltistani — Cardiologist    (doctor login)

Punjab
└── Lahore General Hospital             (hospital-admin login)
    ├── Dr. Ayesha Raza — Internal Medicine     (doctor login)
    └── Dr. Bilal Ahmed — Emergency Medicine    (doctor login)

Sindh
└── Karachi Civic Hospital              (hospital-admin login)
    ├── Dr. Sana Iqbal — General Medicine       (doctor login)
    ├── Dr. Omar Farooqi — Cardiologist         (doctor login)
    └── Dr. Rabia Yousuf — Child Specialist     (doctor login)

Khyber Pakhtunkhwa
└── Peshawar City Hospital              (hospital-admin login)
    ├── Dr. Nadia Khattak — General Medicine        (doctor login)
    └── Dr. Adeel Yousafzai — Orthopedic Surgeon     (doctor login)

Balochistan
└── Quetta Regional Hospital            (hospital-admin login)
    ├── Dr. Bilal Marri — General Medicine       (doctor login)
    └── Dr. Mahnoor Achakzai — Gynecologist      (doctor login)

Islamabad Capital Territory
└── Islamabad Capital Hospital          (hospital-admin login)
    ├── Dr. Usman Farooq — General Medicine      (doctor login)
    └── Dr. Hira Abbasi — Dermatologist          (doctor login)

Azad Jammu & Kashmir
└── Muzaffarabad General Hospital       (hospital-admin login)
    ├── Dr. Faiza Chaudhry — General Medicine    (doctor login)
    └── Dr. Waqas Mughal — Child Specialist      (doctor login)
```

Every hospital in the seed data has its own hospital-admin login, entirely
separate from any of its doctors' logins — DHQ Gilgit's admin account is not
Skardu Civil Hospital's, and neither is Lahore General's.

### Who can log in, and what they can see/do

| Account | Logs in via | Scope |
| --- | --- | --- |
| **Patient** | National ID + OTP (unchanged by this feature) | Their own record, dependents, and appointments only. |
| **Doctor** | `POST /api/auth/doctor/login` (email + password) | Their own patients/appointments only, exactly as before. Belongs to exactly one hospital (`doctors.hospital_id`), but has no admin capability over that hospital. |
| **Hospital admin** | `POST /api/auth/hospital-admin/login` (email + password) | Doctors: only the ones at their own hospital (`hospital_admins.hospital_id`) — list, create, edit specialization, deactivate/reactivate, all scoped server-side to their own `hospital_id` from the signed session, never a request parameter. **Cannot** see or touch another hospital's doctors, and cannot transfer a doctor between hospitals (`hospital_id` isn't accepted as writable input on any hospital-admin route). Patients: can search and fix registration/demographic fields (name, DOB, contact info, blood group, allergies-on-file, etc.) via the same validation as a doctor's edit — but **cannot** see medical history, diagnoses, prescriptions, or the clinical audit trail; that surface is doctor-only, both in the API and the UI. |
| **Analytics analyst** | Analytics app's own login (unchanged — admin/viewer, national scope) | Every region and every hospital, combined, as aggregate counts only — never a name, National ID, or individual record. |

Hospital-admin sessions use the exact same mechanics as doctor sessions
(bcrypt password hashing, a signed JWT in an `httpOnly` cookie, the same
CSRF double-submit-cookie protection) — see `backend/src/lib/auth.ts`. It's
implemented as a peer of doctor auth (its own cookie,
`pulseid_hospital_admin_session`, and its own session type,
`HospitalAdminSession`), not layered on top of it or the analyst pattern.

Deactivating a doctor (`POST /api/hospital-admin/doctors/:id/deactivate`)
never deletes them — their historical records, prescriptions, and
appointments stay exactly as they were. It just blocks future logins
(`findDoctorByEmail` only matches active doctors) and removes them from the
patient-facing "choose a doctor" picker (`listDoctorsForBooking`).

### Seeding the database with this structure

```bash
cd backend
npm run seed              # first run only seeds if the DB is empty
# or, to wipe and reseed from scratch:
node scripts/seed.js --force
```

This creates all 8 hospitals shown in the tree above — one per region, with
two in Gilgit-Baltistan (DHQ Gilgit + Skardu Civil Hospital) as the worked
example — each with 2–3 doctors across different specializations (never an
empty hospital), plus one hospital-admin account per hospital. The seed
script prints every generated hospital-admin email/password to the console
right after the existing doctor/patient credential printout, e.g.:

```
[seed] Hospital-admin logins (one per hospital, password same for every admin in this demo seed):
  - Lahore General Hospital: admin.lahoregeneral@pulseid.dev / hospitaladmin123 (Zainab Malik)
  - DHQ Gilgit: admin.dhqgilgit@pulseid.dev / hospitaladmin123 (Rahat Karim)
  - Skardu Civil Hospital: admin.skarducivil@pulseid.dev / hospitaladmin123 (Bilal Skardu)
  ...
```
(Full list is 8 lines, one per hospital — check your terminal output after
seeding for the exact set, since re-running with `--force` regenerates IDs
but keeps the same emails/hospitals.)

If you're upgrading an existing database that predates this feature (no
`hospital_admins` table, no `doctors.is_active` column), no manual migration
step is needed — `backend/src/lib/db.ts` adds both automatically the next
time the backend starts, defaulting every existing doctor to active.

### How analytics rolls this up

Every analytics aggregate (`backend/src/lib/repo.ts`, the section below
`Analytics — national/regional aggregates`) is built by joining
`medical_records → doctors → hospitals`, so a visit's region always comes
from the hospital where it was recorded — the full chain, not a
region-to-doctor shortcut. On top of the existing province/city rollups
(`getProvinceSummary`, `getRegionSummary`), `getHospitalSummary(region?)`
adds the hospital-level layer: for each hospital, its region/province, its
doctor count, and its visit/patient counts — exposed at
`GET /api/analytics/hospitals?region=<city>` and surfaced in the analytics
dashboard as a drill-down under each region row on the **Regions** page
(`analytics/app/dashboard/regions/page.tsx`).

The same small-cell suppression rule applies at every level: any count below
`MIN_CELL_SIZE` (5) is returned as `null` with `suppressed: true` instead of
an exact number, rendered as `<5` by `SuppressedValue` — this stops someone
from combining a hospital + a rare diagnosis to re-identify a specific
patient, the same principle used at the region/province level.

Empty sets are handled defensively at every level:
- A **region with zero hospitals** — `getHospitalSummary(region)` simply
  returns an empty array (a `LEFT JOIN` starting from `hospitals`, so a
  region that has no hospital rows produces no output rows, never an error).
  The dashboard renders "No hospitals on file for this region." instead of
  an empty or broken table.
- A **hospital with zero visits** — still gets a row: `COUNT(m.id)` over a
  `LEFT JOIN` to `medical_records` with no matches is exactly `0`, which
  `suppress()` renders the same way as any other small cell (`<5`) rather
  than dividing by zero or omitting the hospital.
- Every share/percentage calculation elsewhere in `repo.ts`
  (`getRegionBenchmark`, `getDataQualityReport`, etc.) already guards its
  denominator (`totalRegionVisits > 0 ? … : null`), so a region with no
  visits yet renders `—` instead of `NaN` or a crash.

## Running it locally

You'll need three terminals: backend, frontend, and (optionally) analytics.

### 1. Backend

```bash
cd backend
cp .env.example .env      # edit SESSION_SECRET to any long random string,
                           # and ANALYTICS_SERVICE_KEY if you'll run analytics too
npm install
npm run dev                # seeds the demo database on first run, then starts on :4000
```

Optionally, in a fourth terminal, run the appointment reminder sweep
(without SMTP/Twilio configured it just logs what it would send — see
[Sending real OTP codes by SMS](#sending-real-otp-codes-by-sms) above for
setting those up for real):

```bash
cd backend
npm run reminders                    # runs on a schedule (every 15 min by default)
npm run reminders -- --run-now       # or fire a sweep immediately, for testing
```

Similarly, in a fifth terminal, run the proactive follow-up sweep (see
[Proactive follow-ups](#proactive-follow-ups--scheduled-check-ins-for-high-risk-patients)
above):

```bash
cd backend
npm run followups                    # runs on a schedule (every hour by default)
npm run followups -- --run-now       # or fire a sweep immediately, for testing
```

### 2. Frontend

```bash
cd frontend
cp .env.example .env # SESSION_SECRET must match the backend's exactly
npm install
npm run dev                 # starts on :3000
```

Open **http://localhost:3000**. To try it as an installed app, open it in
Chrome/Edge and use the install icon in the address bar (or the browser
menu → "Install PulseID…" / "Add to Home Screen" on mobile).

### 3. Analytics (optional)

```bash
cd analytics
cp .env.example .env # ANALYTICS_SERVICE_KEY must match the backend's exactly;
                            # set ANALYTICS_ADMIN_EMAIL/ANALYTICS_ADMIN_PASSWORD to
                            # bootstrap your first (admin) analyst account — add more
                            # named accounts later from the "Analysts" page;
                            # set ANTHROPIC_API_KEY to enable the AI tabs (Ask, Alerts
                            # briefing, Reports, Resources) — optional, everything else
                            # works without it
npm install
npm run dev                 # starts on :3100
```

Open **http://localhost:3100** and sign in with `ANALYTICS_ADMIN_EMAIL` /
`ANALYTICS_ADMIN_PASSWORD` (the login form pre-fills the default demo values
— `admin@health.gov` / `change-this-password` — so you can just hit Sign in).
See `analytics/README.md` for more on how it's isolated from the
doctor/patient side, and its own README section on per-analyst accounts,
roles, and the audit log.

> **Note:** `.env`/`.env.local` files are only read once, at process
> startup. If you edit `backend/.env` or `analytics/.env.local` (e.g. to
> change `ANALYTICS_SERVICE_KEY`) while `npm run dev` is already running,
> restart that process — otherwise the analytics dashboard will fail with
> `503 Analytics API is not configured.` even though the file looks correct.
> Also remember `ANALYTICS_SERVICE_KEY` must be byte-for-byte identical in
> both `backend/.env` and `analytics/.env.local`, or every `/api/analytics/*`
> call gets a `401` instead.

### Backend in Docker (recommended for anything beyond local dev)

The backend ships with a multi-stage `Dockerfile` and a `docker-compose.yml`
at the repo root, so it can be run as a container with a persistent volume
for its SQLite data — the frontend still runs separately with `npm run dev`
or your own hosting.

```bash
cp .env.example .env      # set SESSION_SECRET, ANALYTICS_SERVICE_KEY,
                           # ANALYTICS_SESSION_SECRET, ANALYTICS_ADMIN_EMAIL,
                           # ANALYTICS_ADMIN_PASSWORD
                           # (all via `openssl rand -hex 32` except the email/password),
                           # and CORS_ORIGIN
docker compose up --build -d
```

This builds the backend image (installing the native toolchain
`better-sqlite3` needs, then stripping it back out of the final runtime
layer), starts the API on `http://localhost:4000` (override with
`BACKEND_PORT`), seeds the schema on first boot — including safe in-place
migrations for older databases (see `backend/src/lib/db.ts`) — and persists
`/app/data` in a named Docker volume (`pulseid-data`) so the database
survives rebuilds and restarts. It also starts `backend-reminders` (no
exposed port — sweeps for appointment reminders every 15 minutes in the
background, sharing the same `pulseid-data` volume as the API) and
`backend-followups` (no exposed port — sweeps for due proactive follow-up
check-ins hourly, also sharing `pulseid-data`), and builds and starts
`analytics` (on `http://localhost:3100`, override with
`ANALYTICS_PORT`) and `analytics-scheduler` (no exposed port — it just runs
the weekly/monthly bulletin cron in the background), both waiting for the
backend's healthcheck before starting. `analytics` and
`analytics-scheduler` share a `pulseid-analytics-data` volume, so bulletins
the scheduler generates show up in the dashboard's review queue.

Useful commands:

```bash
docker compose logs -f backend               # tail backend logs
docker compose logs -f backend-reminders     # tail reminder-sweep logs
docker compose logs -f backend-followups     # tail follow-up-sweep logs
docker compose logs -f analytics             # tail analytics web logs
docker compose logs -f analytics-scheduler   # tail scheduler logs (bulletin generation runs)
docker compose exec backend node scripts/seed.js --force   # reset + reseed demo data
docker compose down                # stop (keeps the data volumes)
docker compose down -v             # stop AND delete the data volumes
```

Point the frontend at it by setting `NEXT_PUBLIC_API_URL=http://localhost:4000`
in `frontend/.env.local` (or your production API domain). The analytics
service is configured entirely through `docker-compose.yml`/`.env` when run
this way — it doesn't need its own `.env.local` in the Docker path.

### Demo logins

The seed script prints these to your terminal, but for convenience:

| Role    | Credential                                    |
| ------- | ---------------------------------------------- |
| Doctor  | `ayesha.raza@pulseid.dev` / `doctor123`        |
| Hospital admin | `admin.lahoregeneral@pulseid.dev` / `hospitaladmin123` (see the "Regional hierarchy" section above for the full list of 8) |
| Patient | National ID `35202-1234567-1` (OTP shown on screen after "Send code") |
| Analytics (optional service) | `admin@health.gov` / `change-this-password` — pre-filled on the login form, from `ANALYTICS_ADMIN_EMAIL`/`ANALYTICS_ADMIN_PASSWORD` in `analytics/.env.local` |

## Deploying it for real

- Deploy `backend/` (`npm run build && npm run start`) somewhere with a
  persistent disk for the SQLite file — or swap `better-sqlite3` for a
  hosted Postgres later, `lib/repo.ts` is the only file that would need to
  change.
- Deploy `frontend/` (`npm run build && npm run start`) anywhere Next.js
  runs — Vercel, a Node server, a container. PWA installability needs
  **HTTPS** in production (localhost is exempted for local dev) — most
  hosts (Vercel, Netlify, a reverse proxy with Let's Encrypt) handle this
  automatically.
- Set `NEXT_PUBLIC_API_URL` (frontend) to the backend's public URL, and
  `CORS_ORIGIN` (backend) to the frontend's public URL.
- If frontend and backend end up on different top-level domains rather than
  subdomains of the same one, cookies won't be readable by Next.js
  middleware across origins — either put them on `app.yourdomain.com` /
  `api.yourdomain.com` and set `COOKIE_DOMAIN=.yourdomain.com`, or drop the
  middleware-based route guard and check auth via a `/api/me` fetch on each
  protected page instead (the pages already fetch it for the header, so this
  is a small change).
- Set `NODE_ENV=production` on the backend — this switches session cookies
  to `Secure; SameSite=None` automatically, required for HTTPS cross-site
  cookies.
- If you change the app icons, update both `frontend/public/icon-*.png`
  *and* `frontend/public/site.webmanifest` — the manifest is what installers
  actually read.
- Deploy `analytics/` (`npm run build && npm run start`, or its own
  `Dockerfile`) anywhere Next.js runs — it does not need to share a domain,
  cookie, or host with `frontend/`, since it has no shared session with it.
  Set `API_INTERNAL_URL` to the backend's URL reachable from analytics'
  server, and make sure `ANALYTICS_SERVICE_KEY` matches the backend exactly.
  Put this behind its own access control at the network level too (VPN,
  IP allowlist, or SSO in front of it) if it's holding data for a real
  ministry-of-health rollout — the built-in single shared password is meant
  for a small trusted group, not open internet access.

## Scaling this up (SQLite → production database)

Everything above runs on SQLite, which is genuinely fine for a demo or a
single small deployment, but a system meant to sit across every hospital in
a country needs a database built for concurrent writers, replication, and
backups. The intended path: swap `better-sqlite3` for `pg` (or an ORM like
Drizzle/Prisma) inside `backend/src/lib/db.ts` and `lib/repo.ts` — those two
files are the only place the rest of the codebase talks to the database, by
design. Once on Postgres:
- Point the analytics service's queries at a **read replica**, not the
  primary, so a dashboard query can never compete with a doctor saving a
  record.
- Consider moving the heaviest analytics aggregates (`getRegionSummary`,
  `getRegionConditionMatrix` in `lib/repo.ts`) out of live query time
  entirely and into a nightly or streaming rollup table — the routes in
  `server.ts` wouldn't need to change, only what they read from.

## Sending real OTP codes by SMS

By default (`DEMO_MODE=true`), the 6-digit patient login code is returned
directly in the `request-otp` API response so you can test without a real
phone number. For a real deployment, set `DEMO_MODE=false` and configure a
[Twilio](https://console.twilio.com) account:

```bash
TWILIO_ACCOUNT_SID=...
TWILIO_AUTH_TOKEN=...
TWILIO_FROM_NUMBER=+1...
```

`backend/src/lib/sms.ts` sends the code via Twilio's plain REST API (no SDK
dependency needed — just the built-in `fetch`). If `DEMO_MODE=false` and
these vars aren't set, the server logs a loud startup warning, since
patients would otherwise be able to request a code that never arrives.

On top of the existing per-IP rate limit on `request-otp`, each National ID
is also capped at 8 OTP sends per rolling 24 hours (`canSendOtp` in
`lib/repo.ts`) — this stops someone from rotating IPs to keep sending (and
running up SMS charges against) codes for one specific patient.

The same Twilio credentials also power appointment reminders (see
[Appointments](#appointments--request-vs-schedule-are-two-different-roles)
above) — `backend/src/lib/sms.ts` exports a generic `sendReminderSms`
alongside the OTP-specific sender, both going through the same Twilio call.
Reminders can additionally (or instead) go out by email; see `SMTP_*` in
`backend/.env.example` and `backend/src/lib/email.ts`. Configuring neither
just means the reminder scheduler logs what it would have sent instead of
actually sending anything — useful for local dev, not something you'd want
left that way in production.

## Tests

The backend has a small `vitest` suite covering the OTP lifecycle — issuing,
verifying, expiry, wrong-code lockout, and the daily send cap — since that's
the code path gating patient login. Run it from `backend/`:

```bash
npm test
```

It runs against an isolated throwaway SQLite file (via `PULSEID_DB_PATH`),
never the real dev database. A GitHub Actions workflow
(`.github/workflows/ci.yml`) runs this plus both backend and frontend builds
on every push and PR.

## Security notes

- Passwords are hashed with bcrypt; a fixed dummy hash is compared on
  unknown emails so login timing can't be used to enumerate accounts.
- Sessions are short-lived signed JWTs in `httpOnly` cookies — never
  readable by JavaScript, never stored in `localStorage`.
- Auth endpoints, the emergency lookup, and QR-token lookup are all
  rate-limited per IP.
- The emergency QR token rotates on every single use, including patient
  logins that view it — a stale screenshot can never be replayed once it's
  been scanned once.
- Every read of a patient's full record, every edit to a patient's details,
  and every emergency scan is written to an audit log the patient can see
  themselves — access transparency by default, not as an afterthought.
- Role boundaries are enforced server-side, not just hidden in the UI:
  appointment scheduling and patient-detail edits both live behind
  `requireDoctor` middleware, and there is no patient-facing route that
  reaches either capability, by design or by accident. Hospital-admin
  routes (`requireHospitalAdmin`) go further and scope by *identity*, not
  just role: every query/update is filtered to the admin's own
  `hospitalId` from their signed session, so there's no request shape that
  reaches another hospital's doctors, and no route accepts `hospital_id` as
  writable input at all (so a doctor can never be transferred between
  hospitals via this API). The doctor-activity view a hospital admin can
  open is similarly scoped and read-only — built from the same
  `audit_logs` table a patient's own audit trail comes from, filtered to
  one doctor's `actor_id`, and never exposes the diagnoses/notes behind an
  entry, only the action/patient/timestamp.
- Bulk doctor import runs the identical per-row validation as creating a
  doctor one at a time (email/license uniqueness, password length), plus
  a same-file duplicate check the single-create path doesn't need — so a
  CSV can't slip in two rows with the same email that would otherwise both
  individually pass the "does this already exist in the database" check.
- Recurring appointments and waitlist offers both still go through the
  same `requireDoctor`-gated confirm/status route as a one-off
  confirmation — there's no separate, less-guarded path for creating a
  batch of appointments or converting a waitlist entry into one.
- The risk-scoring flag (blood pressure, blood sugar, temperature, heart
  rate → low/mid/high) is deterministic and rule-based, not a trained
  model — every score returns the plain-language `factors` that produced
  it plus a fixed disclaimer, so it can always be explained and audited,
  never treated as an opaque diagnosis.
- Every proactive-follow-up route is ownership-scoped, not just
  role-scoped: a doctor can only create, view status/check-ins for, or
  acknowledge alerts on agents where `doctor_id` matches their own signed
  session (`findFollowupAgentForDoctor`, `acknowledgeFollowupAlert` in
  `lib/repo.ts`), and a patient can only answer a check-in that is both
  `pending` and belongs to their own `patient_id`
  (`findPendingCheckinForPatient`) — there's no request shape that lets
  one doctor see another's follow-up patients, or one patient answer
  someone else's check-in. Risk flagging on a submitted check-in never
  depends solely on the optional AI summary; the deterministic
  threshold-based flag (`flagCheckinResponses`) is what alerting is built
  on, so alerts still work correctly with zero AI provider configured.
- The service worker only caches static, public app-shell assets (icons,
  the manifest, the offline page) — it never intercepts or caches requests
  to the API origin, so an installed app can't accidentally serve someone
  stale medical data while offline.
- The doctor patient-directory CSV export (`/api/patients/export.csv`)
  neutralizes formula injection: any field starting with `=`, `+`, `-`, `@`,
  a tab, or a carriage return is prefixed with a leading `'` so spreadsheet
  apps (Excel, Google Sheets) always render it as plain text instead of
  evaluating it as a formula.
- The backend refuses to start in production unless `PUBLIC_APP_URL` is
  set. Without it, the URL embedded in every patient's emergency QR code
  falls back to the request's `Host` header — which a client can forge — so
  an unset value in production could let someone point freshly-generated QR
  codes at a domain they control.
- A repo-root `.gitignore` keeps `.env` files, `node_modules`, the SQLite
  `data/` directory, and build output out of version control.
- The analytics service is a fully separate trust boundary: it authenticates
  to the backend with a service-to-service key (`ANALYTICS_SERVICE_KEY`),
  never a doctor/patient session, and every route it can call returns
  aggregate counts only — small-cell-suppressed below 5 — never a name,
  National ID, or individual record. It has its own login and session
  secret (`ANALYTICS_SESSION_SECRET`), so an analytics credential and a
  clinical credential can never be swapped for each other, by design.
- `backend`'s `/api/analytics/*` routes refuse every request (503) if
  `ANALYTICS_SERVICE_KEY` isn't set, rather than silently running open.
- The analytics app uses named, bcrypt-hashed analyst accounts (admin or
  viewer role) instead of a single shared password — every bulletin
  approval, AI query, CSV export, and account change is written to that
  app's own audit log with the analyst's identity attached, and admin-only
  actions (approving bulletins, managing analyst accounts, reading the
  audit log) are enforced in the route handler itself, not just hidden in
  the nav. Its login route is rate-limited per IP, and AI queries are
  rate-limited per analyst since each one costs real money against the
  configured provider.

## Project structure at a glance

```
backend/
├── src/
│   ├── server.ts        All API routes, grouped by area (auth, patients,
│   │                     appointments, emergency, reports, audit,
│   │                     hospital-admin, waitlist, risk, follow-ups)
│   └── lib/
│       ├── db.ts         SQLite connection + in-place schema migrations
│       ├── repo.ts       All data-access functions (one place to swap DBs)
│       ├── auth.ts       Session signing/verification, password hashing
│       ├── report.ts     Shared report-building logic (patient + doctor)
│       ├── pdf-report.ts PDF rendering for the medical report
│       ├── sms.ts        Twilio SMS (OTP delivery + appointment reminders
│       │                  + follow-up check-in prompts)
│       ├── email.ts      SMTP email (appointment reminders + follow-up
│       │                  check-in prompts)
│       ├── reminders.ts  24h appointment-reminder sweep logic
│       ├── risk-scoring.ts  Deterministic, rule-based vitals → risk-level
│       │                     scorer (general + maternal contexts)
│       ├── followup-agent.ts  Proactive follow-up sweep, rule-based
│       │                       check-in risk flagging, default question
│       │                       templates per pathology
│       └── ai.ts         OpenRouter/Anthropic wrapper for the hospital-admin
│                          appointment-insights briefing and follow-up
│                          check-in summaries, degrades gracefully
└── scripts/
    ├── seed.js                  Creates the schema + demo data on first run
    ├── reminder-scheduler.ts    Standalone cron process — runs the reminder
    │                             sweep every 15 min via reminders.ts
    └── followup-scheduler.ts   Standalone cron process — runs the
                                  proactive follow-up sweep hourly via
                                  followup-agent.ts

frontend/
├── app/                  Next.js App Router pages, split by role
│   ├── doctor/            dashboard, patients, appointments, calendar, scan,
│   │                       followups
│   ├── patient/            home, appointments, records, dependents, followups
│   ├── hospital-admin/     overview, dashboard (doctors), patients, appointments
│   └── emergency/         public, unauthenticated pages
├── components/           Shared UI, plus doctor/, patient/, hospital-admin/
│                          specific pieces (including StartFollowupForm,
│                          FollowupDashboard, CheckinForm)
├── public/
│   ├── sw.js              Service worker (installability + offline fallback)
│   ├── offline.html        Offline fallback page
│   └── site.webmanifest    PWA manifest (icons, name, install shortcuts)
└── middleware.ts          Verifies the session cookie, guards protected routes

analytics/
├── app/
│   ├── login/              Password-gated analyst login
│   ├── api/
│   │   ├── login, logout    Session issuing/clearing
│   │   ├── trends, forecast Same-origin proxies for TrendExplorer
│   │   ├── query             AI: natural-language question → aggregate-only answer
│   │   ├── alerts/narrate    AI: plain-language briefing over active alerts
│   │   ├── bulletins         List/generate weekly+monthly bulletins (AI + charts)
│   │   ├── bulletins/[id]/review  Approve/reject a pending bulletin
│   │   └── resources/suggest AI: "worth a look" suggestions, never decisions
│   └── dashboard/          Overview, Regions, Map, Trends, Benchmark, Alerts,
│                            Query, Reports (Bulletins UI), Resources, Quality
├── components/            Nav, KPI cards, suppressed-value display, TrendExplorer,
│                           RegionMap (Leaflet), BulletinCharts, AiSummaryBox,
│                           QueryBox, shared ui.tsx
├── scripts/
│   └── scheduler.ts        Standalone cron process — generates bulletins on a
│                            schedule via the same code the manual button uses
├── lib/
│   ├── api.ts               Backend aggregate-data client (used by app + scheduler)
│   ├── ai.ts                 OpenRouter/Anthropic wrapper, degrades gracefully
│   ├── auth.ts               Analyst session signing/verification
│   ├── bulletin-db.ts         Analytics' own SQLite store for generated bulletins
│   ├── bulletin-generator.ts  Shared "gather data → ask AI → store" bulletin logic
│   └── geo.ts                 City → lat/lng lookup for the Map tab
├── Dockerfile               The web app (build + serve)
├── Dockerfile.scheduler     Just runs scripts/scheduler.ts, no HTTP server
└── middleware.ts           Guards every route except /login and /api/login
```

## Built for the hackathon, built to actually be used

Split services, a shared but minimal trust boundary (one secret, one
cookie), rate limiting, rotating QR tokens, server-enforced role boundaries,
a full access-transparency log, and honest offline behavior aren't hackathon
decoration — they're what it actually takes for people to trust a system
that holds their medical history. That was the goal here: not just a demo
that works on stage, but a codebase that's honest about what a real
medical-records product needs — including, now, a way to see the shape of
the nation's health data without ever exposing the people behind it.
