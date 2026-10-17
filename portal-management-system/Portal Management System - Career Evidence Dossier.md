# Portal Management System — Career Evidence Dossier

**Subject:** Abraham Akintola
**Project:** Portal Management System (repo `portal-management-system`; Composer package `bouesti/bportal`)
**Production instances audited:** EKSU (Ekiti State University) — `eksuportal.eksu.edu.ng`; BOUESTI — `portal.bouesti.edu.ng`, `ptpportal.bouesti.edu.ng`; and **College of Education, Ikere-Ekiti** — the predecessor institution that became BOUESTI, served by its own database (`coeikere-db`) from Dec 2020
**Audit dates:** EKSU 13 October 2026; BOUESTI 14 October 2026
**Evidence sources:** repository source, Git history (155 commits, Feb 2023 – Oct 2026), recovered modernization log, Azure Resource Manager (read-only), Azure Monitor metrics, EKSU and BOUESTI production MySQL (read-only `SELECT`/aggregate queries only)

> **Read-only guarantee.** All database access used `SET SESSION TRANSACTION READ ONLY` with `SELECT`/aggregate queries exclusively. No `INSERT`, `UPDATE`, `DELETE`, DDL, migration or administrative command was executed. No credentials, connection strings, keys or personally identifiable information appear anywhere in this document or its companion files.

---

## 1. Executive Summary

The Portal Management System is a large legacy PHP/MySQL student and staff portal that runs the end-to-end academic administration of Nigerian universities — admissions, student registration, result processing, transcripts and certificates, fee collection, hostel allocation, CBT examinations, e-learning and student-union e-voting. It is a genuinely live system across two deployments: in the 31 days to 13 October 2026 the two instances served **22.56 million HTTP requests**, and their activity logs recorded **82,176 distinct users** over 30 days (EKSU 53,143, BOUESTI 29,033).

**My role is co-founder and one of two engineers who built, own and operate this system.** It began as an undergraduate project, was adopted by the university on graduation, and has run in production for approximately seven years — EKSU from September 2019, BOUESTI from 2021 — with no third engineer at any point. That two-person scope spans product, development, database administration, cloud infrastructure, deployment and institutional support across two universities.

Within that, the work this dossier verifies in file-level detail is the **2026 cloud migration and storage platform engineering**: moving the system off cPanel/Apache shared hosting onto Azure and off local disk onto object storage, without breaking a 5,400-file legacy codebase that was still serving live traffic. I am the sole author of every infrastructure, storage and deployment file involved.

**A note on how to read the attribution here.** This repository's Git history begins in February 2023 with a 10,000-file bulk import — three years *after* the system was already live. A commit-count split drawn from it describes a late window, not the system's authorship. Specific features committed by my co-founder in that window (CSRF/XSS hardening, two-factor auth, e-voting, payment flows) are attributed to them throughout; that is a split between two co-founders of the same product, not a boundary between an owner and a contributor. §4 sets out the three distinct levels of claim and the evidence behind each.

Both instances were migrated by the same work in the same window, and the evidence below covers both.

**The strongest verified evidence:**

| Metric | Value | Source |
|---|---|---|
| Production HTTP requests served, 31 days to 13 Oct 2026 (EKSU) | **14,092,342** | Azure Monitor |
| Same, both instances combined | **22,561,348** | Azure Monitor |
| HTTP 5xx rate across those requests (EKSU) | **0.0076%** (1,064 responses) | Azure Monitor |
| Health-check availability, 7 days to 13 Oct 2026 | **100.00%** | Azure Monitor, probe `/health.php` (which I authored) |
| Production file references migrated from local disk to Azure Blob | **675,590** (EKSU 571,040 + BOUESTI 104,550) across 24 columns | Both production databases |
| Migration completeness across all mapped columns | **98.70%** — 8,886 legacy values remain in 4 low-volume mappings | Both production databases |
| Objects in the shared Blob account (both schools) | **~1,197,452**, **~85.8 GB** | Azure Monitor |
| Storage-abstraction adoption in the codebase | **263 call sites across 91 PHP files** | repository grep |
| Peak single-day request volume (EKSU) | **1,239,422** on 12 Oct 2026 | Azure Monitor |

**Clearest business impact:** two university portals that previously depended on a single shared-hosting box now run on managed Azure App Service with a managed MySQL Flexible Server and a durable object store, deployed by an automated GitHub Actions pipeline, with ~85.8 GB of sensitive student documents moved out of ephemeral application disk into access-tiered Blob containers. The migration was completed while the system stayed live through an admissions cycle, a fee-collection cycle and a student-union election.

---

## 2. Project Overview

### What the system does

A multi-tenant-by-deployment academic portal. One codebase, deployed as two independent Azure Web Apps, each pointed at its own database and its own storage containers:

| Instance | Azure Web App | Hostnames | Database |
|---|---|---|---|
| EKSU | `eksu-app-prod` | `eksuportal.eksu.edu.ng` | `eksu-db` |
| BOUESTI | `bouesti-app-prod` | `portal.bouesti.edu.ng`, `ptpportal.bouesti.edu.ng` | `bouesti-db` (+ `coeikere-db` via host routing) |

Functional surface, evidenced by the directory structure and the 153-table production schema:

- **Admissions** — application intake, O'level/UTME result capture, document upload, admission letters
- **Student records** — registration, curriculum/level progression, screening document verification
- **Results** — raw score capture, result processing, transcripts, graduating lists
- **Transcripts & certificates** — request workflow with payment, approval chain, document attachments
- **Fees** — Remita and Credo payment gateway integration, receipts, clearance gates
- **Staff** — profiles, payroll, promotion, memos, roles/permissions, publications
- **Hostel** — hall/room/bed allocation
- **CBT** — computer-based testing (3,472 tests, 127,083 questions in production)
- **E-books / virtual learning** — content distribution with comments and view tracking
- **E-voting** — student-union elections with identity verification and 2FA

### Scale (verified, EKSU production, 13 Oct 2026)

| Entity | Count |
|---|---|
| Database size | 6.30 GB across 153 tables |
| Student records | 135,067 |
| Staff records | 2,900 across 193 department codes |
| Admission records | 149,663 |
| Raw score rows (result processing) | 5,517,117 |
| Remita payment transactions | 1,856,617 |
| Student level/progression rows | 812,228 |
| Activity-log events | 846,971 |
| Transcript requests | 52,457 |
| Certificate requests | 68,287 |
| Faculties / departments / courses | 22 / 303 / 7,783 |

**BOUESTI production (14 Oct 2026):** 5.20 GB across 151 tables; 33,099 student records; 747 staff; 65,964 admission records; 1,335,391 raw score rows; 271,451 Remita transactions; 8,471,215 activity-log rows; 22 faculties / 218 departments / 5,517 courses. The two schemas are the same shape — 151–153 tables — because they are the same codebase.

### Codebase

5,427 PHP files, ~1.17 million lines excluding `vendor/`. 284 files include the central `server.php` bootstrap. 5,039 raw `mysqli_query()` call sites across 341 files. This is a genuinely legacy procedural PHP application, originally written for PHP 7 on Apache with `.htaccess` rewriting, now running on PHP 8.3 on nginx.

---

## 3. Business Problem and Context

**The problem (VERIFIED from `PLANS.md`, `AGENTS.md`, `.htaccess`, and the recovered modernization log):**

The system was built for and ran on cPanel/Apache shared hosting. That brought four concrete constraints:

1. **Routing depended on `.htaccess`.** Every legacy URL in the application is extensionless (`/login`, `/students/index`) and resolved by an Apache `RewriteRule ([^\.]+)$ $1.php`. Azure's PHP Linux image runs nginx and ignores `.htaccess` entirely — so a lift-and-shift produces a site where every internal link 404s.
2. **User-uploaded files were written to local disk.** Student passports, signatures, screening documents, admission certificates, transcripts, payment evidence — all written as plain filenames into `uploads/`, `images/`, `ebooks/`. On App Service, local disk is not a durable system of record, and the historical archive was estimated in `PLANS.md` at 100–300 GB.
3. **Credentials were hardcoded in source.** Database credentials in `server.php`, Remita/Credo payment keys in `module/r_value.php`.
4. **MySQL dialect drift.** The source was MariaDB 10.6; the target is Azure MySQL 8.0, which enforces TLS, rejects `CAST(x AS INT)`, and applies `only_full_group_by` — all of which the old server tolerated.

### How the engagement began — USER-REPORTED

The first draft inferred a "managed-hosting engagement" from the Azure layout. The actual origin, as reported by the author and summarised here for the record:

The system began as an undergraduate portfolio project built by two students, with no intention of selling it. During their degree, the university's existing portal failed twice. The first failure (externally managed) left students unable to pay fees or access results. Its replacement, run by a single individual connected to the institution, was discontinued after a financial-misconduct incident involving substantial sums. The university then attempted to build a replacement in-house with academic staff and final-year students, and separately took an admissions-only system from a third party, neither of which produced a working portal.

On graduating, the two authors approached the university with what they had already built. It was accepted, adapted to the institution's curriculum, results and fee structures, and taken into production — initially unpaid, then on a nominal stipend during national service, scaling over seven years to a commercial arrangement. The same codebase was later taken on by a second institution.

**Why this matters technically, not just biographically.** It explains constraints that are otherwise unreadable from the code:

- **The trust deficit was the binding constraint.** Following a fraud incident, a two-person team of recent graduates had to earn access to student financial data and document storage. This is the most plausible explanation for the system's heavy `activity_log` instrumentation (846,971 and 8,471,215 rows) and for institution-controlled admin dashboards — auditability was a precondition of being allowed to operate at all.
- **It explains the responsiveness model.** The reported ability to turn around institutional change requests within hours or days is consistent with a two-person team holding full-stack ownership and no release committee — and with the absence of staging discipline noted in §11.
- **It explains the architecture's shape.** Local-server hosting was the institution's initial requirement, abandoned when the provided hardware proved unusable, which is how the system reached cPanel and ultimately Azure.

### The second and third institutions — how BOUESTI came about

The first draft treated `coeikere-db` as a routing quirk of the BOUESTI deployment. It is not. It is a **third production deployment, for a separate institution**, and it carries its own history.

The reported sequence: roughly a year or two after EKSU went live, the team won a contract to build a portal for the **College of Education, Ikere-Ekiti**. That college subsequently gained approval to become a university — **Bamidele Olumilua University of Education, Science and Technology (BOUESTI)** — at which point a new instance was stood up for the university, while the college instance continued serving its existing cohort through to completion.

**This is VERIFIED from production, and the sequence matches exactly:**

| Evidence | Value |
|---|---|
| `coeikere-db` — a complete, separate 151-table database | 0.25 GB |
| Its first logged activity | **11 Dec 2020** |
| Students served | **11,785** (entry years back to 2012 — legacy import) |
| Staff | 278 |
| Raw scores | 132,243 |
| Payment transactions | 41,707 |
| Activity-log events | 498,788 |
| Peak year | **2022** — 206,754 events, 14,384 distinct users |
| Activity profile 2023 → 2026 | 118,333 → 52,973 → 22,340 → 1,457 — a cohort winding down |
| BOUESTI instance first activity | **7 May 2021**, five months after the college instance |
| `setup.siteName` in `coeikere-db` today | *"Bamidele Olumilua University of Education, Science and Technology, Ikere-Ekiti"* — the same institution, renamed |

That decay curve is the strongest single piece of evidence: it is exactly what a **parallel-run institutional transition** looks like — existing students finishing on the legacy instance while new intake goes to the new one, over roughly four years. Running two instances side by side through an institution's change of status is real operational work, and the first draft missed it entirely.

**What is and is not claimable.** That the team built and ran the College of Education's portal, and that the college became BOUESTI while they served both, is **VERIFIED**. That the portal *enabled* the upgrade to university status is **USER-REPORTED and not verifiable** — accreditation decisions turn on many factors. The defensible framing is that the system was in place through the transition and served both institutions across it, not that it caused the outcome.

**Evidence status:** the narrative above is USER-REPORTED and cannot be verified from code. The *technical* consequences drawn from it are inferences. The production timeline it implies (live from 2020) is independently VERIFIED — see §4.

**Handling note:** the predecessor fraud incident is PRIVATE. It concerns a real, identifiable individual and an institution. It must not appear in public portfolio, LinkedIn or article content in any form that could identify either party — see §21.

---

## 4. My Role and Contribution

### Founding context — USER-REPORTED, materially corroborated

**This is not a codebase I joined. I am one of its two authors and co-owners.** The system began as an undergraduate side project built by me and one collaborator, was offered to the university on graduation, and has been built, adapted, operated and owned by the two of us ever since — **in production at EKSU since September 2019**, roughly seven years as of 2026, with no third engineer at any point.

This account is **user-reported**. I did not derive it from the repository, and an employer cannot verify it from the code alone. But four independent pieces of evidence corroborate it, and none contradict it:

| Corroborating evidence | What it shows | Level |
|---|---|---|
| Only **two** author email addresses appear in the entire Git history | No third engineer has ever committed | VERIFIED |
| EKSU `admission` records from **8 Sept 2020**; successful payments tagged to session **2019** | The system was live in production ~3 years before this repository existed | VERIFIED |
| BOUESTI `activity_log` continuous from **7 May 2021** (8.47M rows) | Second institution live from 2021 | VERIFIED |
| First commit (15 Feb 2023) is titled `new_respitory` and lands **10,000 files in one commit** | A bulk import of an already-mature codebase, not the start of development | VERIFIED |
| **1,822,938 EKSU records tagged to academic sessions before 2019** — 1,543,811 raw scores, 220,164 level records, 58,963 student records, sessions configured back to 1990 | Decades of pre-existing institutional academic history were migrated *into* this system, which only happens at the start of a replacement engagement | VERIFIED |

**Production lifespan: approximately seven years at EKSU (September 2019 – present) and five at BOUESTI (May 2021 – present), continuous and current.** This was an unresolved evidence gap in the first draft.

**A precision point on the EKSU start date.** The system went live in **September 2019** (user-reported). The earliest *independently verifiable* timestamp anywhere in the EKSU production database is **9 July 2020**, and the earliest payment data is tagged to 2020. That gap is explained, consistently with the evidence, by what the first phase actually involved:

- **Sept 2019 – early 2020:** loading legacy results, curricula and student data, and running academic records. The tables used in that phase (`rawscore`, `studentslevel`, `students`, `curriculum`) are tagged by **academic session**, not by row timestamp, so they leave no datable trace — and their session tags run back to 1990 because they hold imported institutional history, not system activity.
- **2020:** payment processing was integrated directly, after the institution approved it. Before that, students paid through the provider's own portal and the system did not see the transactions. This is why `fees_remita_transaction` begins in 2020 and not 2019.

So the 2020 dates in this dossier mark **when payments and ticketing were added**, not when the system went live. Treat "September 2019" as USER-REPORTED, and "in production by July 2020 at the latest" as VERIFIED.

### Evidence method — and its hard limit

Git authorship across the 155 commits in this repository:

| Author | Commits | Period | Focus (from commit subjects and file paths) |
|---|---|---|---|
| Co-founder (`princeaden1`, committing as Adekunle / PRINCE ADEN) | 121 | Feb 2023 – Sep 2026 | CSRF/XSS hardening; 2FA; e-voting; payments; result processing |
| **Abraham Akintola (me)** | **33** | **6 Aug – 10 Oct 2026** | **Azure migration, storage abstraction, blob migration, CI/CD, DB connection layer, routing, health** |

**Read this table with its limitation, which is severe.** This repository's history begins on 15 Feb 2023 with a 10,000-file bulk import. The system had already been in production at EKSU since 2020. **The Git history therefore covers roughly the last three of seven years and cannot speak to original authorship at all.** A commit-count split drawn from it measures who pushed changes in a late window — not who built the system.

The first draft of this dossier concluded from that table that I "was not the application's author." **That conclusion was wrong**, because it treated a partial window as the whole record. It has been corrected throughout. What the table still supports, and what I continue to claim narrowly, is the *2026 Azure and storage work* — that is independently verifiable from the files themselves.

My commits in this window touched 1,409 file-paths with ~40,596 insertions and ~38,947 deletions (figures inflated by a vendored Composer dependency add/revert — see Appendix).

### VERIFIED personal contribution

Each item below is tied to a file whose entire Git history is mine, or to commits authored by me.

**1. Azure App Service runtime compatibility for legacy Apache routing.**
`azure-nginx-default.conf` and `startup.sh` — both files, sole author. The nginx config reimplements the `.htaccess` extensionless rewrite as an nginx `try_files` → named-location fallback, adds deny rules for dotfiles and `server.php`, and raises FastCGI timeouts to 3600s for the portal's slow report pages. `startup.sh` is the App Service startup hook: it probes the four possible nginx default-config paths in the PHP image, copies the config in, runs `nginx -t`, and reloads — logging to `/home/LogFiles`. This is wired into both Web Apps' startup command (`bash /home/site/wwwroot/startup.sh`), verified live via Azure RM.

**2. Database connection hardening for Azure MySQL Flexible Server.**
Commit `b5560b0c` added configurable `DB_PORT`, `DB_CONNECT_TIMEOUT`, `DB_SSL_MODE` and `DB_SSL_CA` to both the mysqli and PDO paths in `module/Database.php`; `f7528ec9` enforced SSL certificate verification and improved connect-error logging. The layer auto-detects Azure (`str_contains($host, '.mysql.database.azure.com')`) and defaults to TLS on that basis — so the same code runs locally without TLS and on Azure with it, without a config change. *(The `Database` class itself was created by my colleague; the Azure/TLS/port/timeout capability is mine.)*

**3. Secrets externalisation to environment / App Settings.**
Commit `881bbc7f` externalised database configuration and `2dfdd315`/`4cf3cede`/`3fb285a1` moved connection strings to the Azure instance. `.env` is gitignored; `.env.example` documents 50+ variable names with no values. Live Azure App Settings confirm 52 settings on `eksu-app-prod` and 32 on `bouesti-app-prod`, including all gateway keys and storage credentials.

**4. The storage abstraction layer — `module/Storage.php` (1,007 lines, sole author).**
This is the most substantial single piece of engineering I contributed. It is a complete, dependency-free Azure Blob Storage client plus a legacy-compatibility layer:

- **Three-tier visibility model** — `private` / `protected` / `public` containers, resolved from environment, so student documents never land in a publicly readable container.
- **Hand-rolled Azure SharedKey request signing** (`signBlobRequest`) — constructs the canonical string-to-sign and HMAC-SHA256 signature for PUT/GET/DELETE/HEAD against the Blob REST API, with canonicalized `x-ms-*` headers and a canonicalized resource path. No Azure SDK dependency.
- **Hand-rolled service SAS generation** (`signedBlobReadUrl`) — time-boxed (`AZURE_STORAGE_URL_TTL`), HTTPS-only, read-only signed URLs with a 300-second clock-skew window, for browser-direct image preview.
- **A `blob://visibility/path` reference scheme** stored in the database instead of absolute paths, so the DB never encodes a hostname or a filesystem layout.
- **Dual-mode operation** — `STORAGE_DRIVER=local|blob`, auto-detected from whether storage credentials are present. Every read path falls back to the legacy local file if a row still holds a plain filename, which is what made an incremental migration possible.
- **An authenticated streaming proxy** (`storage.php` at the web root, also mine) for files that must not be exposed via SAS.
- **`materializeMany()` — concurrent blob fetch via `curl_multi`** with a configurable in-flight window (default 12), per-request temp-file caching, and per-item failure isolation. The inline comment states the reason: sequential materialisation costs one HTTPS round trip plus a TLS handshake per blob, which made bulk exports (hundreds of passports into one ZIP/PDF) scale linearly into the request timeout.

**5. Codebase-wide adoption of that layer.**
Ten procedural wrapper functions (`storage_preview_url`, `storage_file_url`, `storage_store_upload`, `storage_local_path`, `storage_zip_add_file`, `storage_materialize_many`, …) were introduced so ~1.17M lines of legacy procedural code could adopt object storage without an OO refactor. Current adoption: **263 call sites across 91 files**, spanning student and staff profiles, screening, admissions, transcripts, certificates, e-books, virtual learning, publications, memos, e-voting and the PDF/ZIP generation paths.

**6. The bulk migration tool — `scripts/migrate_legacy_files_to_blob.php` (619 lines, sole author).**
A mapping-driven migrator covering ~14 named mappings over 24 columns across 11 tables, each declaring its table, key, legacy directory, target visibility and blob prefix. Supports `--only=<mapping>` for batched runs and defaults to **dry-run**; `--execute` is required to write. It reports what will migrate, what is already blob-backed, and which referenced files are missing from disk. Commits `2e7d22cb`, `71fe87d1`, `6c90c26f` and `35f288c5` record dry-run, diagnostic and execution passes.

**7. CI/CD.**
`.github/workflows/azure-app-service.yml`, sole author. A matrix build deploying the same commit to both Web Apps in parallel (`fail-fast: false`, so one school's failure doesn't block the other), with `rsync --delete --exclude-from='.webapp-deploy-ignore'` producing a trimmed package. `.webapp-deploy-ignore` (mine) strips `.git`, vendor docs/tests/examples, SQL dumps, logs and local archives from the deployed artefact.

**8. Health check endpoint.**
`health.php`, sole author (commit `f0a556c6`). Wired as the App Service health probe path on `eksu-app-prod`, verified live.

**9. Repository hygiene.**
Commit `cbf3b356` removed 85 user-uploaded files that had been committed to the repository and added `uploads/*` to `.gitignore`.

### Attributed to my co-founder within the Git window

These were committed by my co-founder between Feb 2023 and Sep 2026, and I do not claim them individually: CSRF token implementation (`module/csrf.php`); the XSS sink audit and fixes; two-factor authentication (`module/two_factor.php`); bcrypt password migration and rotation (`module/password.php`); the e-voting module and its identity verification (`module/vote_identity.php`); the MySQL-dialect compatibility sweeps (`CAST ... AS SIGNED`, `only_full_group_by`, `$EntryYear` quoting); the Dropify patch; lazy `$conn2`/`$pconn` loading; and the `Database` class.

**What this list does and does not mean.** It is a *within-window* split between two co-founders of the same product, not a boundary between an owner and a contributor. On a two-person team that has run a system for seven years, both people are accountable for all of it; the split simply records who last touched what. It would be wrong to read this list as "the parts Abraham had nothing to do with" — it covers three of seven years and says nothing about the system's original design or construction.

**Three levels of claim, which should be kept distinct:**

| Level | What it covers | Evidence |
|---|---|---|
| **Co-authorship and co-ownership of the whole system** | Design, build, institutional adaptation, seven years of operation | USER-REPORTED, corroborated (two committers only; production from 2020; bulk-import first commit) |
| **The 2026 Azure migration and storage platform** | `Storage.php`, migration tool, nginx/startup, CI/CD, DB TLS layer, health endpoint | VERIFIED — sole author of every one of those files |
| **Specific features committed by my co-founder** | CSRF, 2FA, e-voting, password rotation | VERIFIED as theirs within the window |

**Qualification on method:** a meaningful portion of this project's recent work on both sides was AI-assisted (the recovered modernization log contains session entries signed "Claude"). For the 2026 window, commit authorship remains the honest boundary — it reflects who owned, directed, reviewed and shipped each change.

---

## 5. System Architecture

```
                       ┌──────────────────────────────────────────┐
   Students / Staff    │  eksuportal.eksu.edu.ng                  │
   Applicants          │  portal.bouesti.edu.ng  ptpportal…       │
   (browser, mostly    └──────────────────┬───────────────────────┘
    mobile)                               │ HTTPS only, TLS ≥ 1.2
                                          ▼
   ┌───────────────────────────────────────────────────────────────────────┐
   │ Azure App Service — Linux, PHP 8.3, Premium P0v3 (1 worker each)      │
   │ Region: South Africa North.  Resource group: portal-management-system │
   │                                                                       │
   │  ┌──────────────────────┐        ┌──────────────────────┐             │
   │  │  eksu-app-prod       │        │  bouesti-app-prod    │             │
   │  │  health: /health.php │        │                      │             │
   │  └──────────┬───────────┘        └──────────┬───────────┘             │
   │             │                               │                         │
   │  startup.sh ──▶ installs azure-nginx-default.conf ──▶ nginx -t, reload│
   │                 (restores Apache extensionless routing)               │
   │                                                                       │
   │  nginx ──▶ PHP-FPM :9000 ──▶ server.php (bootstrap, session,          │
   │            rate limit, CSRF scope, per-host DB routing)               │
   │                     │                                                 │
   │            module/bootstrap.php ──▶ Composer autoload                 │
   │                     ├──▶ Dotenv::safeLoad()  (App Settings in prod)   │
   │                     ├──▶ Database.php   (singleton + lazy 2nd + PDO)  │
   │                     ├──▶ csrf.php / password.php                      │
   │                     └──▶ Storage.php   (blob | local)                 │
   └──────┬──────────────────────────┬─────────────────────┬───────────────┘
          │ mysqli, TLS enforced     │ Blob REST,          │ HTTPS
          │                          │ SharedKey HMAC      │
          ▼                          ▼                     ▼
 ┌─────────────────────┐   ┌──────────────────────┐  ┌──────────────────┐
 │ Azure DB for MySQL  │   │ portalfilestorage01  │  │ Remita           │
 │ Flexible Server 8.0 │   │ StorageV2, LRS, Hot  │  │ Credo            │
 │ Standard_E2ds_v5    │   │ ~1.20M blobs ~85.8GB │  │ SMTP (PHPMailer) │
 │ MemoryOptimized     │   │                      │  └──────────────────┘
 │ 25 GB, 7-day backup │   │ {eksu|bouesti}-      │
 │ HA: disabled        │   │   portal-private     │   Browser-direct reads
 │                     │   │   portal-protected   │   use short-lived SAS;
 │ eksu-db   (6.30 GB) │   │   portal-public      │   sensitive reads go
 │ bouesti-db          │   └──────────────────────┘   through storage.php
 │ coeikere-db         │                              (authenticated proxy)
 └─────────────────────┘

 Deployment:  GitHub `main` ──▶ Actions matrix (EKSU ∥ BOUESTI)
              ──▶ rsync --delete --exclude-from=.webapp-deploy-ignore
              ──▶ azure/webapps-deploy@v3 (publish profile per school)

 Observability: Application Insights connection string set on both apps;
                App Service health probe; nginx/PHP logs to /home/LogFiles
```

**Component notes (all VERIFIED against live Azure or source):**

- **Tenancy** is deployment-level, not row-level: one codebase, two Web Apps, two databases, two sets of containers. Within BOUESTI there is an additional per-hostname DB route in `server.php` (`portalcoe`/`ptpportal` → `DB_NAME_COE`).
- **Session state** is PHP's default local handler. `server.php` carries a prominent warning I verified in source: file-based sessions must not be pointed at the Azure Files `/home` mount, because PHP's files handler enforces a uid check that fails on that CIFS mount. This is the reason both apps run a single worker — see §13.
- **Rate limiting** is a per-session, per-path counter in `server.php` (20 requests / 10 s) with an explicit allow-list for heavy print/report endpoints, plus a crude user-agent block.
- **Security controls**: HTTPS-only, TLS ≥ 1.2, FTPS-only publishing, CSRF enforcement scoped to `/students/`, `/admissions/`, `/cert_transs/`, `/staffs/`, `/votes/` and toggleable via `CSRF_ENFORCE`, bcrypt passwords with configurable cost, email-based 2FA with throttling (79,825 throttle rows in production), and e-voting identity verification with lockout.

---

## 6. Main Workflows

**Upload → storage → render (the path I rebuilt).** A student submits a passport photo. `storage_store_upload()` sanitises the filename (random prefix + slug), detects MIME type, and — when `STORAGE_DRIVER=blob` — PUTs it to `{school}-portal-private/student/passports/<name>` with a SharedKey-signed request, returning `blob://private/student/passports/<name>`. That string, not a path, is what the database stores. On render, `storage_preview_url()` returns either a short-lived SAS URL (browser fetches Azure directly) or a base64-encoded reference to `storage.php`, which authenticates the session and streams the blob. For server-side PDF/ZIP generation, `storage_local_path()` / `storage_materialize_many()` pull blobs to temp files — the latter concurrently — and register them for cleanup. If a row still holds a bare legacy filename, every one of those paths falls back to the local file. That fallback is what allowed the migration to be incremental rather than a cutover.

**Bulk migration.** `migrate_legacy_files_to_blob.php` walks each mapping, finds rows whose column holds a non-`blob://` value, locates the matching file in the declared legacy directory, uploads it, and updates the column. Dry-run by default; `--only=` for batching.

**Deployment.** Push to `main` → GitHub Actions matrix → rsync-trimmed package → `azure/webapps-deploy@v3` to both Web Apps → App Service runs `startup.sh` → nginx config installed and validated → legacy routes resolve.

**Workflows authored by my colleague** (listed for completeness, not claimed): admission application → document upload → screening → admission letter; course registration → fee assessment → Remita/Credo payment → clearance → result; transcript/certificate request → payment → approval chain → issue; e-voting → identity verification → 2FA → ballot.

## 7. Technology Stack

| Layer | Technology |
|---|---|
| Language / runtime | PHP 8.3 (Linux, App Service built-in image); Composer, `^8.0` constraint |
| Web server | nginx + PHP-FPM (custom site config) |
| Database | Azure Database for MySQL Flexible Server 8.0.21, `Standard_E2ds_v5`, MemoryOptimized, 25 GB, 7-day backup, HA disabled |
| Object storage | Azure Blob Storage — `portalfilestorage01`, StorageV2, Standard_LRS, Hot; 6 containers (3 visibility tiers × 2 schools) |
| Compute | Azure App Service Premium P0v3, 1 worker per app, South Africa North |
| Frontend | jQuery/AJAX SPA shells with PHP partials; DataTables; Dropify; Bootstrap |
| Libraries | PHPMailer 6.9.3, TCPDF 6.9.1 (65 files), PhpWord 1.3.0, vlucas/phpdotenv ^5.6, PhpSpreadsheet |
| Payments | Remita (425 source references), Credo (23); residual Flutterwave/Paystack/Interswitch references |
| CI/CD | GitHub Actions → `azure/webapps-deploy@v3`, matrix over two schools |
| Observability | Application Insights; App Service health probe; file logs to `/home/LogFiles` |
| Secrets | Azure App Settings (52 on EKSU, 32 on BOUESTI); `.env` gitignored locally |

---

## 8. Production Evidence and Statistical Analysis

Every figure below states what was measured, over what period, from what source, and its limitations.

### 8.1 Traffic and reliability — Azure Monitor

**Measured:** HTTP request and status-code counts, 1-day grain.
**Period:** 12 Sep 2026 – 13 Oct 2026 (31 daily data points). This window is set by Azure Monitor's retention at 1-day grain, not by the system's lifespan; the apps have been live since 8 Aug 2026.
**Calculation:** sum of daily `Total` aggregation per metric.

| Metric | EKSU | BOUESTI | Combined |
|---|---|---|---|
| Requests | 14,092,342 | 8,469,006 | **22,561,348** |
| HTTP 2xx | 11,622,269 | 7,715,365 | 19,337,634 |
| HTTP 3xx | 917,484 | 218,470 | 1,135,954 |
| HTTP 4xx | 1,551,525 | 533,393 | 2,084,918 |
| HTTP 5xx | 1,064 | 1,776 | **2,840** |
| 5xx rate | 0.0076% | 0.021% | 0.013% |
| Mean requests/day | 454,592 | 273,194 | 727,786 |
| Peak day | 1,239,422 (12 Oct) | — | — |

**Limitations:** requests include static assets, so this is not a page-view or user count. The 4xx figure (11% on EKSU) is not analysed here — it plausibly includes 404s for missing legacy assets and 403s from the rate limiter, but I did not break it down and will not claim a cause.

**Availability.** App Service `HealthCheckStatus` for `eksu-app-prod` averaged **100.00%** over the 7 days to 13 Oct 2026, probing `/health.php`.

**Latency.** `AverageResponseTime` for `eksu-app-prod`, daily, 29 Sep – 12 Oct 2026: median ≈ **0.19 s**, with 10 of 14 days under 0.29 s. Two outliers: 2.05 s on 29 Sep (election day) and 3.52 s on 12 Oct (the 1.24M-request peak). This is a credible single-worker profile under spiky load. *(DERIVED from daily averages; I did not have percentile data.)*

### 8.2 Blob storage migration — both production databases

**Measured:** for each mapped column, rows whose value begins `blob://` versus rows holding a non-empty, non-blob legacy value.
**Dates of measurement:** EKSU 13 Oct 2026; BOUESTI 14 Oct 2026.
**Calculation:** `SUM(col LIKE 'blob://%')` and `SUM(col IS NOT NULL AND col <> '' AND col NOT LIKE 'blob://%')`.

| Column group | EKSU blob refs | BOUESTI blob refs | Legacy remaining (either) |
|---|---|---|---|
| `admission.imageUrl` | 143,712 | 61,994 | 0 |
| `transc_certificate` (4 columns) | 205,057 | 5,196 | 0 |
| `students.passport` | 92,239 | 29,685 | 0 |
| `transcript` (4 columns) | 69,900 | 16 | 0 |
| `students.signature` | 52,629 | 7,474 | 0 |
| `students` screening (8 columns) | 4,576 | 72 | 0 |
| `staffs.passport` / `.signature` | 2,903 | 112 | 0 |
| `admission` document set (7 columns) | 21 | 0 | 0 |
| `leave_of_absence.attachedFile` | 3 | 0 | **6,258** |
| **Total migrated** | **571,040** | **104,550** | — |

**Combined: 675,590 migrated file references across both instances.**

**Correction to an earlier draft of this dossier.** An initial pass reported "zero legacy paths remaining" without having queried the legacy side of `leave_of_absence.attachedFile`. It is in fact the largest unmigrated mapping. The accurate position is below.

**Unmigrated mappings — the honest incompleteness:**

| Mapping | EKSU legacy | BOUESTI legacy | Blob-backed |
|---|---|---|---|
| `leave_of_absence.attachedFile` | 5,906 | 352 | 3 |
| `ebooks.fileUrl` | 2,478 | 41 | 1 |
| `ebooks_comment.file_attach` | 0 | 107 | 0 |
| `message.messageImage` | 2 | 0 | 0 |
| **Total legacy remaining** | **8,386** | **500** | — |

**Migration completeness: 675,590 / 684,476 = 98.70%.** All nine high-volume document mappings — student and staff identity media, admission documents, screening uploads, transcript and certificate request attachments — are fully migrated on both instances with zero legacy values. The 8,886 remaining values sit in four low-volume mappings that the migration script supports but which were never executed, and which keep working through the legacy-filesystem fallback. §14 explains why that fallback is precisely the reason they stalled.

**Azure-side corroboration:** `portalfilestorage01` holds ~**1,197,452 blobs** / ~**85.77 GB** (Azure Monitor, 12 Oct 2026, both schools combined), and recorded **1,966,226 storage transactions** in the 30 days to 13 Oct 2026. The account is shared, so this figure cannot be split per school from metrics alone.

**Migration date is directly evidenced on both instances.** Rows carry an `appliedDate` of exactly **2026-08-15**, and the correlation with blob references is perfect in both databases:

| Instance | Table | Rows stamped 2026-08-15 | Of those, holding blob refs | All other rows holding blob refs |
|---|---|---|---|---|
| EKSU | `transcript` | 37,346 | 37,346 (100%) | 2,926 of 15,111 (19.4%) |
| EKSU | `transc_certificate` | 48,770 | — | — |
| BOUESTI | `transc_certificate` | 1,324 | 1,324 (100%) | 0 of 121 (0%) |

That is 87,440 request records across both instances migrated in a single day's execution. See §14 for why this is simultaneously strong evidence and a data-integrity finding.

### 8.3 Adoption and usage — both production databases

**Activity log** (`activity_log`, 846,971 rows, authenticated user actions with timestamps):

| Measure | Value | Period |
|---|---|---|
| Distinct users, 2026 session | **42,613** (42,410 students + 203 staff) | session `2026` |
| Distinct users, 2025 session | 48,329 | session `2025` |
| Peak daily active users | **19,769** | 12 Oct 2026 |
| Second-highest DAU | 14,430 | 22 Sep 2026 |
| Mean DAU, 13 Sep – 12 Oct 2026 | ≈ 7,600 | 30 days |
| Most recent event at audit time | 13 Oct 2026, 01:54 | live |

**BOUESTI activity log** (`activity_log`, 8.47M rows — a longer-retained log than EKSU's):

| Measure | Value | Period |
|---|---|---|
| Distinct users, 30 days | **29,033** | 14 Sep – 14 Oct 2026 |
| Distinct users, 90 days | 39,096 | 16 Jul – 14 Oct 2026 |
| Peak daily active users | **16,075** | 24 Aug 2026 |
| Most recent event at audit time | 14 Oct 2026, 22:36 | live |

**Comparable, date-based figures for both instances** (the session-tagged counts above are not comparable, because BOUESTI's `activity_log.session` column has not been rolled over to `2026` — it still reads `2025` for current events, which is itself a small data-quality finding):

| Window | EKSU | BOUESTI | Combined |
|---|---|---|---|
| Distinct users, 30 days | 53,143 | 29,033 | **82,176** |
| Distinct users, 90 days | 61,492 | 39,096 | 100,588 |

**Limitation:** "distinct users" counts distinct `activityUser` identifiers that generated a logged event. Not every page view is logged, so this understates reach. The combined figure may double-count anyone holding accounts at both institutions, which is unlikely but unverified.

**Payments** (`fees_remita_transaction`, 1,856,617 rows, all sessions since ~2020):

| Measure | Value |
|---|---|
| Successful transactions | **1,551,157** (83.55%) |
| Pending | 211,424 (11.39%) |
| Error | 91,794 (4.94%) |
| Distinct paying students (successful) | **223,663** |
| Aggregate value of successful transactions | ≈ **₦48.78 billion** |

**BOUESTI payments** (`fees_remita_transaction`, 271,451 rows): **238,827 successful (87.98%)**, **61,758 distinct paying students**, aggregate successful value ≈ **₦16.73 billion**. Errors 8.70%, pending 3.29%. Combined across both instances: **1,789,984 successful transactions**, **285,421 distinct payers**, ≈ **₦65.51 billion**.

**Important limitations.** (a) This is the full historical table, most of it predating the Azure migration and predating my involvement — it evidences *the system's* throughput, not my work. (b) `paymentAmount` is a `varchar`; the sum is `CAST(... AS DECIMAL)`. (c) **I only summed `Successful` rows deliberately** — the `Pending` rows contain values that are clearly not naira amounts (e.g. a 2021 "total" of ~2×10¹³), so that column holds reference numbers or junk for non-successful rows. Any payment figure quoted from this table must be restricted to `paymentStatus='Successful'`.

**E-voting** (`tbvotedetails`, the September 2026 student-union election):

| Measure | Value |
|---|---|
| Ballots cast | 7,188 |
| Distinct voters | 5,730 |
| Voting window | 26 Sep 23:50 – 29 Sep 12:59, 2026 |
| Busiest hour | 1,809 ballots, 29 Sep 09:00–10:00 |
| Candidates / positions | 393 / 190 |

This correlates with the Azure traffic spike on 28–29 Sep (882,510 and 962,046 requests) and the 2.05 s response-time outlier — the platform carried an election under load without a 5xx spike. **BOUESTI's `tbvotedetails` is empty**, so e-voting is an EKSU-only production workload; do not generalise the election claim to both institutions.

**Other verified production counts:** CBT — 3,472 tests, 127,083 questions, 374,799 results. Email verification — 25,916 of 135,067 student records verified (19.2%); a rollout in progress, not a completed one. Two-factor throttle table — 79,825 rows.

---

## 9. Business Impact

Stated at the level the evidence supports, and no further.

**VERIFIED — delivered identically to both institutions.** The same work landed at both schools, but each has its own measured outcome. §23 gives the full per-institution split; this is the impact view.

| Impact | EKSU | BOUESTI |
|---|---|---|
| Migrated to managed Azure App Service + MySQL + Blob | Yes — `Running` | Yes — `Running` |
| Production traffic sustained, 31 days | 14,092,342 requests | 8,469,006 requests |
| HTTP 5xx rate over that window | 0.0076% | 0.021% |
| File references moved off local disk | 571,040 | 104,550 |
| Legacy values still on the filesystem path | 8,386 | 500 |
| Distinct users served, 30 days | 53,143 | 29,033 |
| Health probe wired (`/health.php`, my endpoint) | Yes — **100.00%** over 7 days | Yes — **99.99%** over 7 days |
| Mean response time, 7 days to 13/14 Oct | 0.19 s (median of dailies) | 0.496 s |

- Sensitive documents (passports, signatures, O'level and birth certificates, transcripts, payment evidence) are stored in `private`/`protected` containers rather than web-reachable directories, and served through an authenticated proxy or a time-boxed signed URL — **on both instances, in separate container sets** (`eksu-portal-*` and `bouesti-portal-*`), so neither school's documents are reachable with the other's references.
- Deployment moved from manual upload to an automated **two-target** GitHub Actions pipeline on push to `main`. This is the one impact that is genuinely indivisible: a single pipeline serves both schools, and neither could have it without the other.
- Credentials moved out of source control into Azure App Settings — 52 settings on EKSU, 32 on BOUESTI, separately scoped per school.

**DERIVED.**
- Application servers became disposable with respect to user data. Because uploads no longer live on the instance, App Service can be redeployed, restarted or scaled without data loss — which is the precondition for horizontal scaling, not the achievement of it (the apps still run one worker; see §13).

**INFERRED — do not state as fact.**
- Cost and operational-overhead changes versus the previous cPanel hosting. No billing or prior-hosting data was available.
- Any effect on student-facing experience, administrative effort or staff time. No before/after measurement exists.

**Asymmetries worth knowing before you split these.** The two schools are not two copies of one outcome:

- **EKSU is roughly 5.5× BOUESTI by migrated file volume** (571,040 vs 104,550) and ~4× by student records (135,067 vs 33,099), but only ~1.7× by traffic. BOUESTI is proportionally the busier deployment per student.
- **E-voting is EKSU-only.** BOUESTI's `tbvotedetails` is empty. The election-capacity story belongs to EKSU alone.
- **BOUESTI carries a third tenant.** Its `server.php` routes the `portalcoe`/`ptpportal` hostnames to a separate `coeikere-db`, which EKSU has no equivalent of. BOUESTI is the more complex routing case.
- **BOUESTI runs ~2.6× slower per request** (0.496 s vs 0.19 s mean) on the same plan size, despite lower traffic. Cause not established — do not speculate in a CV; it is logged in §22.
- **EKSU's transcript/certificate workflow is far heavier** — 120,744 combined requests vs 3,223 — which is why the migration-day timestamp damage is overwhelmingly an EKSU problem (86,116 of the 87,440 affected rows).

**Qualitative, not quantified.** The system's *functional* impact — digitising admissions, registration, results, transcripts, fees and elections for a combined ~168,000-record student body — is real but largely predates and is independent of my contribution. I claim the platform migration, not the digitisation.

---

## 10. Significant Engineering Challenges

**Restoring Apache routing semantics on nginx without touching 5,400 files.** Every internal link in the application is extensionless. The options were rewriting URLs across the codebase, or reproducing the rewrite at the web-server layer. The second is correct but App Service's PHP image gives you no supported hook for a custom nginx site config — the config path differs between image versions and is overwritten on restart. The solution had to be a startup script that discovers the path at runtime across four candidates, validates with `nginx -t` before reloading, and logs to a durable location so a failed startup is diagnosable. Difficulty here is operational, not algorithmic: the failure mode is a site that returns 404 for every page with no obvious cause.

**Writing an Azure Blob client by hand in legacy PHP.** The Azure PHP SDK is effectively unmaintained and would have dragged a dependency tree into a PHP 8.3 app that still has PHP-7-era assumptions. `Storage.php` instead implements the Blob REST contract directly: the SharedKey canonical string-to-sign (a strict, whitespace-sensitive 12-newline format where a single wrong field yields an opaque 403), canonicalized `x-ms-*` header ordering, RFC-3986 path encoding that must encode the path but not the `/` separators, and a separate service-SAS string-to-sign with its own field order and API-version pinning. Two distinct signing schemes, both unforgiving, both failing with the same uninformative error.

**Making an incremental migration survivable.** A straight cutover would have broken every page the moment the first row was updated. The design decision that made it safe: store an opaque `blob://` reference and make *every* read path — preview, download, ZIP, PDF, delete — fall back to the legacy local file when the value isn't a blob reference. That one invariant meant the database could be migrated table by table, column by column, while the site stayed up.

**Bulk export performance against object storage.** Moving files off local disk converts a `fopen()` into an HTTPS round trip with a TLS handshake. A page that builds a ZIP of several hundred student passports goes from milliseconds-per-file to ~100ms-per-file, which pushes it into the FastCGI timeout. `materializeMany()` solves this with `curl_multi` and a bounded in-flight window, streaming each response straight to disk. The subtle part is the scheduling loop: a job that fails before reaching curl (temp-file or signing failure) consumes its slot without entering the active set, so the loop must refill from the top of the iteration rather than only on completion — otherwise a batch of such failures spins with nothing in flight. That bug is explicitly commented in the source because it is non-obvious and easy to reintroduce.

**Multi-tenancy without a tenancy model.** The application has no tenant column. Isolation is achieved by deploying the same commit to two Web Apps with different App Settings, different databases and different container names — plus a per-hostname database route retained inside `server.php` for the COE sub-tenant. It is a deployment-time tenancy model and it works, but it means any tenant-specific behaviour lives in environment variables rather than in code.

---

## 11. Architectural Decisions and Trade-offs

**Decision: nginx config installed by a startup script, not a custom container.**
*Problem:* App Service's built-in PHP image ignores `.htaccess`. *Alternatives:* rewrite all URLs; ship a custom Docker image. *Reasoning:* URL rewriting touches thousands of files with no test suite; a custom image means owning PHP patching and base-image updates for a team that wanted managed hosting. *Trade-off:* the startup script is coupled to App Service's internal nginx layout and could break on an image update — mitigated by probing four paths, validating with `nginx -t`, and logging. *Result:* extensionless routes work on both apps; the startup command is live in production.

**Decision: a `blob://` reference scheme in the database, rather than storing URLs.**
*Problem:* what goes in the `passport` column once files move to Blob. *Alternatives:* full HTTPS URL; container-relative path. *Reasoning:* a URL bakes the account name, endpoint and container into every row, making account migration or endpoint changes a data migration. A bare path loses the visibility tier. The `blob://visibility/path` form encodes only what the application needs and resolves account/endpoint/container from environment at read time. *Trade-off:* values are not directly usable by anything outside the app. *Result:* 571,040 production rows carry these references; the storage account, endpoint and container names are all still environment-configurable.

**Decision: dual-mode storage with permanent legacy fallback.**
*Problem:* migrating hundreds of thousands of rows while the site serves live traffic. *Alternatives:* big-bang cutover; a compatibility shim to be removed later. *Reasoning:* `PLANS.md` explicitly warns "do not switch all uploads to Blob without first supporting legacy reads". *Trade-off:* the fallback is permanent complexity in every read path, and `ebooks.fileUrl` shows the risk — a mapping that still works via fallback has no pressure to be migrated, so it hasn't been. *Result:* a safe incremental migration; some residual legacy debt.

**Decision: three containers by exposure level, prefixed per school.**
*Problem:* the application mixes public brand images with private identity documents. *Reasoning:* `AGENTS.md` requires treating student/staff/admission/transcript documents as sensitive by default. *Trade-off:* visibility must be declared at every upload site; a wrong declaration silently over-exposes. *Result:* six containers live; private/protected content is served via SAS or authenticated proxy, never by direct container access.

**Decision: hand-rolled signing rather than the Azure SDK.**
*Trade-off:* I own the correctness and the API-version pinning (`2021-12-02` for REST, `2023-11-03` for SAS); an SDK upgrade path doesn't exist. Justified by avoiding a heavy, poorly maintained dependency in a legacy app — but this is the component most deserving of a test suite, and it has none.

**Decision: matrix deployment to both schools from one `main` push.**
*Trade-off:* no per-school release isolation — a bad commit reaches both universities simultaneously. `fail-fast: false` limits only partial-failure blast radius, not bad-code blast radius.

**Decision: exclude the EKSU `activity_log` from the Azure migration.**
*Problem:* after nearly seven years the EKSU audit log had grown very large, and migrating it would have carried its full history into Azure MySQL storage — which is paid, provisioned capacity on a 25 GB server already holding 6.30 GB of live data. *Decision:* leave it behind and keep the historical log on the old cPanel host as a cold fallback; the table in Azure starts fresh from the migration date. *Reasoning:* it is append-only log data with no transactional dependency, consulted rarely and never on a hot path, so it is the single largest thing that can be dropped from a migration without affecting application behaviour. *Trade-off, stated honestly:* the live system no longer holds its own pre-August-2026 audit trail. For a system whose auditability was a precondition of institutional trust (§3), answering "what happened in 2022?" now requires retrieving an offline copy rather than querying production — and that copy's retention is only as good as the old host's. BOUESTI took the opposite path and kept its full history (8,471,215 rows back to May 2021), so the two instances now differ in audit depth. *Result:* EKSU's `activity_log` holds 846,971 rows from 15 Aug 2026 onward; the historical log is retained off-cloud. **This was a deliberate cost/scope decision, not data loss** — an earlier draft of this dossier wrongly flagged it as possible migration damage.

**Decision (reversed): staging slot, then direct deploy.** Commit `5993941b` (16 Aug) introduced a staging deployment slot; `d082b638` (10 Oct) removed it. The commit messages give no reason, so the reasoning is **UNKNOWN / REQUIRES CONFIRMATION**. This is worth recovering — it is a real release-engineering trade-off I made and currently cannot defend from evidence.

**Decision (reversed): Redis-backed sessions.** Commit `62b5805f` (16 Aug) added Redis sessions with graceful fallback via Predis; `3008db2a` reverted it the next day. The reason is not recorded. The problem it addressed is documented in `server.php`: PHP's file session handler fails on the Azure Files `/home` mount with "Session data file is not created by your uid", so cross-instance sessions need Redis or a DB handler. The revert is why both apps still run a single worker.

---

## 12. Problems Solved

Presented in the Situation / Problem / Investigation / Solution / Result / Evidence form useful for interviews. Problems 1–3 are the engineering described in §4 and §10, framed as incidents; 4–5 are additional.

**1 — Every page 404s after lift-and-shift.** *Situation:* the app deployed and the homepage loaded, but every internal link failed. *Problem:* `.htaccess` extensionless rewriting is invisible to nginx. *Investigation:* traced the Apache rule and confirmed App Service serves PHP via nginx + PHP-FPM with a default site config whose path varies by image version. *Solution:* `azure-nginx-default.conf` reproducing the rewrite as `try_files $uri $uri/ @legacy_php_route`, installed by `startup.sh` which probes four candidate paths, validates with `nginx -t`, reloads, and logs to `/home/LogFiles`. *Result:* `/index`, `/login`, `/students/index`, `/staffs/index` all resolve; the startup command is live on both apps. *Evidence:* commit `dd9eb9ce`; live App Service config; `PLANS.md` records this as completed.

**2 — Azure MySQL rejects the connection.** *Situation:* the app could not connect after the database moved to Flexible Server. *Problem:* Azure enforces TLS by default; the legacy code opened a plain mysqli connection on port 3306 with no timeout. *Solution:* `Database::connect()` gained `DB_PORT`, `DB_CONNECT_TIMEOUT`, `DB_SSL_MODE` and `DB_SSL_CA`; TLS is enabled when `DB_SSL_MODE=require` *or* when the host matches `.mysql.database.azure.com`, so local development needs no config change. `MYSQLI_CLIENT_SSL` is set and `MYSQLI_OPT_CONNECT_TIMEOUT` prevents indefinite hangs. *Result:* both apps connect over TLS to MySQL 8.0.21. *Evidence:* commits `b5560b0c`, `f7528ec9`.

**3 — User uploads on ephemeral application disk.** *Situation:* ~85 GB of student documents written as plain filenames into `uploads/`, with the database storing bare filenames. *Problem:* App Service local disk is not a durable system of record. *Investigation:* traced every upload point, the DB field written, and every render/download/ZIP/PDF consumer — `AGENTS.md` warns specifically that a mechanical find/replace breaks this quietly. *Solution:* the storage layer, wrappers, 263 call-site migrations and the bulk migrator described in §4, with permanent legacy fallback. *Result:* **675,590 references migrated across both production databases — 571,040 at EKSU and 104,550 at BOUESTI — 98.70% of all mapped values**, with every high-volume document mapping fully migrated on both. 8,886 values remain in four low-volume mappings. *Evidence:* commits `dd42c24b` → `35f288c5`; production SQL counts from both databases; Azure Monitor storage metrics.

**4 — Dropify previews broke once URLs carried query strings.** *Situation:* after the cutover, file-upload widgets showed a generic file icon instead of the existing image. *Problem:* proxy and SAS URLs carry `?ref=…` / `?sv=…`; Dropify's `cleanFilename()` keeps the query string, so `getFileType()` reads the extension as `jpg?ref=…` and `isImage()` returns false. *Solution:* patch `cleanFilename()` to strip `?`/`#` in both `dropify.js` and `dropify.min.js` — one change covering all 57 Dropify pages (23 with `data-default-file`). *Evidence:* recovered modernization log, 2026-08-11; commit `186cecba`. **Attribution: committed by my colleague** — included because it is a direct downstream consequence of my storage change.

**5 — Bulk exports timing out against object storage.** Covered in §10; solved by `materializeMany()`. *Evidence:* commit `a61a1b8f`; `Storage.php:434–586`.

## 13. Security, Reliability and Operational Considerations

**Implemented and verified:** HTTPS-only with TLS ≥ 1.2 and FTPS-only publishing on both apps; TLS-enforced database connections; three-tier blob visibility with private/protected content served only via authenticated proxy or time-boxed SAS; all secrets in Azure App Settings with `.env` gitignored; deployment package stripped of `.git`, SQL dumps, logs and vendor test/doc directories; 7-day automated database backups; health probe at `/health.php` with `WEBSITE_HEALTHCHECK_MAXPINGFAILURES` configured; Application Insights connection string set on both apps; per-session rate limiting in `server.php`.

**Honest weaknesses in the current state — these are real and I would raise them in an interview:**

- **No high availability.** MySQL Flexible Server has `highAvailability: Disabled`. Both App Service plans run **one worker**, `alwaysOn: false`. A single instance failure is an outage.
- **Horizontal scaling is blocked by session state.** Sessions use PHP's local file handler. Scaling to two workers would break logins, because there is no shared session store — the Redis work that would have fixed this was reverted. This is the single most important open item.
- **`AZURE_STORAGE_ACCOUNT_KEY` is a long-lived account key in App Settings.** Managed Identity with RBAC would remove the shared secret entirely and is the obvious next hardening step.
- **The storage account is shared between both universities,** isolated only by container naming. The account key grants access to both.
- **No automated tests anywhere in the repository.** The hand-rolled SharedKey and SAS signing code — the component where a subtle error is both most likely and hardest to detect — has no test coverage.
- **`.git` is 261 GB locally** because user uploads were historically committed. Commit `cbf3b356` stopped the bleeding but did not rewrite history.
- **Backup retention is 7 days** with no evidence of a tested restore.

---

## 14. Lessons Learned

**A bulk `UPDATE` rewrote business timestamps, and nobody noticed for two months.** `transcript.appliedDate` and `transc_certificate.appliedDate` are declared `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`. The blob migration's `UPDATE … SET imageUrl = 'blob://…'` therefore silently reset the application date on every row it touched. The evidence is unambiguous, and it replicates across both databases:

| Instance | Table | Rows stamped 2026-08-15 | Holding blob refs | Other rows holding blob refs |
|---|---|---|---|---|
| EKSU | `transcript` | 37,346 | 37,346 (100%) | 2,926 of 15,111 (19.4%) |
| EKSU | `transc_certificate` | 48,770 | — | — |
| BOUESTI | `transc_certificate` | 1,324 | 1,324 (100%) | 0 of 121 (0%) |

The original application dates for **87,440 request records across two institutions** are gone from those columns — 86,116 at EKSU and 1,324 at BOUESTI. The BOUESTI correlation is the cleaner proof (100% vs 0%), because that instance had no prior blob-backed rows to muddy it.

This is the most valuable lesson in the project. The migration script was carefully built — dry-run by default, `--only=` batching, missing-file reporting — and it still destroyed data, because the review focused on *what the script writes* and not on *what the schema does when you write*. The general rule I would now apply: before any bulk `UPDATE` on a legacy schema, enumerate every `ON UPDATE CURRENT_TIMESTAMP` and every trigger on the target tables, and either exclude those columns or pin them explicitly in the `SET` clause. It also argues for doing data migrations against a restored copy first and diffing, rather than validating only the column you intended to change.

*(This is a finding, not a repair. Per the audit's read-only constraint I made no change. It is worth telling the data owner: a backup from before 15 Aug 2026 would still contain the original dates, but 7-day retention means that window has long closed — recovery, if any, would depend on the pre-migration cPanel database dump.)*

**Legacy compatibility is a feature, not debt — until it removes the pressure to finish.** The local-file fallback is what made a live incremental migration possible. It is also precisely why `ebooks.fileUrl` is still 100% legacy two months later: nothing breaks, so nothing forces completion. A migration with a fallback needs an explicit completion tracker, or it stalls at 95%.

**Moving to object storage changes the performance model, not just the storage layer.** Every `fopen()` becomes an HTTPS round trip. Code that was fine iterating files sequentially is not fine iterating blobs sequentially. I would now treat "find every loop that touches more than a handful of files" as a mandatory step of any disk-to-object migration, planned up front rather than discovered through timeouts.

**Platform compatibility is where lift-and-shift migrations actually fail.** Not the database and not the code, but `.htaccess`, TLS enforcement, `CAST(x AS INT)`, `only_full_group_by`, session storage on a CIFS mount, and PHP 8's decision to throw where PHP 7 returned `false`. Each was individually small; collectively they were most of the migration.

**Two reverts I cannot currently explain are themselves a lesson.** Redis sessions and the staging slot were both added and removed within weeks, with commit messages that record the *what* and not the *why*. Those decisions are now unrecoverable from the repository, and one of them (sessions) is still the system's main scaling blocker. Decision records cost minutes and would have preserved both.

---

## 15. Verified Career Claim Bank

> Evidence levels: **VERIFIED** (direct), **DERIVED** (calculated from verified data), **STRONGLY SUPPORTED**, **INFERRED**, **UNKNOWN**.

**C1.** Migrated two live Nigerian university portals from cPanel/Apache shared hosting to Azure App Service (Linux, PHP 8.3) with Azure Database for MySQL Flexible Server, keeping both systems serving production traffic throughout.
*VERIFIED* — commits `6b723272`, `dd9eb9ce`, `3fb285a1`, `b5560b0c`; live Azure RM state (both apps `Running`). *Period:* Aug–Oct 2026. *Qualification:* I was the migration engineer, not the author of the application being migrated.

**C2.** The platform I migrated served **22.56 million HTTP requests across both instances in 31 days** (14.09M EKSU, 8.47M BOUESTI) at a **0.013% HTTP 5xx rate**.
*VERIFIED* — Azure Monitor, 12 Sep – 13 Oct 2026. *Qualification:* request counts include static assets; the 31-day window is Azure's retention limit at 1-day grain, not the system's lifespan.

**C3.** Designed and implemented a 1,007-line Azure Blob Storage abstraction layer with no SDK dependency, including hand-rolled SharedKey HMAC-SHA256 request signing and service-SAS generation, three-tier container visibility, and permanent legacy-filesystem fallback.
*VERIFIED* — `module/Storage.php`, sole author across 7 commits.

**C4.** Migrated **675,590 production file references across 24 database columns and 11 tables, on two independent production databases**, from local filesystem paths to Azure Blob references — **98.70% of all mapped values**, with zero legacy values remaining in every high-volume document mapping.
*VERIFIED* — EKSU (13 Oct 2026) and BOUESTI (14 Oct 2026) production MySQL aggregate queries. *Qualification:* 8,886 values remain legacy across four low-volume mappings (`leave_of_absence`, `ebooks`, `ebooks_comment`, `message`) that the migration tool supports but which were never executed. Quote the 98.70% figure with that caveat, not "zero legacy remaining".

**C5.** The resulting object store holds **~1.2 million blobs / ~85.8 GB** and handled **1.97 million storage transactions in 30 days**.
*VERIFIED* — Azure Monitor, `portalfilestorage01`. *Qualification:* shared across both schools.

**C6.** Migrated **87,440 transcript and certificate request records across both institutions** to object storage in a single day's execution on 15 Aug 2026, using a mapping-driven, dry-run-by-default migration tool I built.
*VERIFIED* — EKSU 37,346 transcript + 48,770 certificate rows and BOUESTI 1,324 certificate rows, all timestamped 2026-08-15 and 100% correlated with blob references; `scripts/migrate_legacy_files_to_blob.php`, sole author.

**C7.** Drove codebase-wide adoption of the storage abstraction across **263 call sites in 91 files** of a ~1.17-million-line legacy procedural PHP codebase, using procedural wrapper functions so no OO refactor was required.
*VERIFIED* — repository grep, 13 Oct 2026.

**C8.** Restored Apache extensionless URL routing on nginx for a 5,400-file legacy codebase by authoring a custom nginx site config and an App Service startup hook that discovers the image's config path at runtime, validates with `nginx -t` and reloads — avoiding both a mass URL rewrite and a custom container image.
*VERIFIED* — `azure-nginx-default.conf`, `startup.sh`, sole author; live startup command on both apps.

**C9.** Hardened the database layer for Azure MySQL Flexible Server with enforced TLS, configurable port, connect timeout and CA path, auto-detecting Azure by hostname so local development required no configuration change.
*VERIFIED* — commits `b5560b0c`, `f7528ec9`; `module/Database.php:38–90`.

**C10.** Built a GitHub Actions matrix pipeline deploying one commit to two independent Azure Web Apps in parallel with per-target publish profiles and an rsync-trimmed deployment package.
*VERIFIED* — `.github/workflows/azure-app-service.yml`, `.webapp-deploy-ignore`, sole author.

**C11.** Eliminated hardcoded database and payment-gateway credentials from source control, moving them to Azure App Settings (52 settings on the EKSU app, 32 on BOUESTI).
*VERIFIED* — commits `881bbc7f`, `2dfdd315`; live App Settings name listing; `.env` gitignored, `.env.example` value-free.

**C12.** Reduced bulk-export latency against object storage by replacing sequential blob fetches with a bounded-concurrency `curl_multi` pipeline (default 12 in flight) with per-item failure isolation and per-request temp caching.
*VERIFIED* as implemented (`Storage.php:434–586`, commit `a61a1b8f`). *The performance improvement itself is INFERRED* — no before/after measurement exists. **Do not quote a speedup figure.**

**C13.** Authored the application health endpoint now used as the Azure App Service health probe, which reported **100.00% availability over the 7 days to 13 Oct 2026**.
*VERIFIED* — `health.php`, sole author, commit `f0a556c6`; live `healthCheckPath`; Azure Monitor `HealthCheckStatus`.

**C14.** The platform carried a student-union election — **7,188 ballots from 5,730 voters over ~48 hours, peaking at 1,809 ballots per hour** — alongside a traffic peak of 962,046 requests/day, with no 5xx spike.
*VERIFIED* (production `tbvotedetails`, Azure Monitor, 26–29 Sep 2026). *Qualification:* **the e-voting application was built by a colleague**, and this ran on the EKSU instance only — BOUESTI's `tbvotedetails` is empty. My claim is platform capacity and reliability, not the feature.

**C21.** Built and operated the student portal for a **third institution** — the College of Education, Ikere-Ekiti — from **December 2020**, serving **11,785 students, 278 staff, 132,243 raw scores and 41,707 payment transactions**, and ran it in parallel with the new BOUESTI university instance through the institution's change of status, winding the legacy cohort down over roughly four years.
*Evidence level:* **VERIFIED** — `coeikere-db` queried directly, 15 Oct 2026: first activity 11 Dec 2020; peak 2022 (206,754 events, 14,384 distinct users); decay 2023→2026 of 118,333 → 52,973 → 22,340 → 1,457 events; BOUESTI instance first activity 7 May 2021. *Qualification:* the claim that the portal *enabled* the college's upgrade to university status is user-reported and **not** verifiable — accreditation turns on many factors. Claim the parallel-run transition, which is fully evidenced; do not claim causation.

**C22 — CONTEXT ONLY, NOT A CAUSAL CLAIM.** EKSU has been publicly ranked among Nigeria's leading state universities during the period this system has served it: **3rd best state university (Webometrics, 2021)**, later cited as **2nd best state university**, and in 2025 the Ekiti State Government described it as the leading state university in Nigeria, citing 12th of 270 Nigerian universities.
*Evidence level:* the **rankings are VERIFIED** from public sources (see §24). **Any link between those rankings and this system is UNVERIFIED** — it rests on verbal remarks reported by institutional staff, with no document, citation or assessor finding behind it. Rankings are multi-factor and no methodology isolates a student portal's contribution. **Use this as period context only** — see §16 for the exact safe wording and why the causal version would damage you.

**C20.** Migrated **1,822,938 historical academic records** into the system at EKSU — 1,543,811 raw scores, 220,164 student level/progression records and 58,963 student records tagged to academic sessions predating 2019, with session configuration back to **1990**.
*Evidence level:* **VERIFIED** — EKSU production MySQL, session-tag aggregation, 15 Oct 2026. *Qualification:* counts records whose academic session predates 2019; it establishes that decades of institutional history were imported, but not the exact date of import. BOUESTI has effectively **no** legacy import (1,283 of 1.34M raw scores predate 2020), so this is an EKSU-specific achievement — do not state it for both schools.

**C18.** Co-authored and co-own a production academic portal that has run continuously for **approximately seven years** (EKSU from September 2019, BOUESTI from May 2021) with a **two-person engineering team and no third engineer at any point**.
*Evidence level:* **STRONGLY SUPPORTED** — user-reported, corroborated by five independent signals: only two author emails in the entire Git history; 1,822,938 EKSU records tagged to pre-2019 academic sessions; EKSU production timestamps from 9 July 2020; BOUESTI activity log continuous from 7 May 2021; and a first commit that bulk-imports 10,000 files, proving the repository post-dates the system. *Qualification:* **"in production by July 2020" is VERIFIED; the September 2019 start is user-reported** — the 2019–mid-2020 phase used session-tagged tables with no row timestamps and predates payment integration, so it leaves no datable trace (§4). Original authorship likewise cannot be proven from this repository. Present as co-founder/co-owner, not "sole author".

**C19.** Have operated that system as one of two engineers responsible for **everything** — product, development, database administration, cloud infrastructure, deployment, security and institutional support — across two universities simultaneously.
*Evidence level:* **STRONGLY SUPPORTED** — the two-committer fact is VERIFIED; the breadth of ownership is user-reported but consistent with the repository containing no CODEOWNERS, no team process, no test suite and no separation between application and infrastructure work.

**C16.** Ran the same migration and platform work against **two independent production deployments** — separate databases (6.30 GB / 153 tables and 5.20 GB / 151 tables), separate blob containers, separate App Service plans — from a single codebase and a single deployment pipeline.
*VERIFIED* — both databases queried directly; `.github/workflows/azure-app-service.yml` matrix; live Azure RM state. *Period:* Aug–Oct 2026.

**C17.** The two instances together served **82,176 distinct users in 30 days** and hold **1,789,984 successful payment transactions** across their production history.
*VERIFIED* — both `activity_log` tables (14 Sep – 14 Oct 2026) and both `fees_remita_transaction` tables. *Qualification:* payments are full historical totals predating my involvement; the user figure may double-count anyone with accounts at both institutions.

**C15.** Identified, from production data, that the bulk migration silently rewrote ~86,000 business timestamps via `ON UPDATE CURRENT_TIMESTAMP`, and established the schema mechanism and 100% correlation with migrated rows.
*VERIFIED* — `information_schema` column definition plus correlation query. *This is a self-reported defect and its root-cause analysis — it is strong interview material precisely because it is honest.*

---

## 16. CV Achievement Candidates

### High-confidence, quantified

- Migrated two live university portals (168,000+ student records across 11.5 GB of MySQL) from cPanel/Apache to Azure App Service and Azure Database for MySQL Flexible Server, now serving **22.5M+ HTTP requests and 82,000+ distinct users per month at a 0.013% 5xx rate** and 100% health-probe availability.
- Designed and shipped a 1,000-line Azure Blob Storage layer with hand-rolled SharedKey and SAS signing (no SDK), then migrated **675,000+ production file references across 24 columns and 11 tables on two independent production databases** — 98.7% of all mapped values, with every high-volume document mapping fully migrated and no downtime.
- Moved **~85.8 GB / ~1.2M user-uploaded documents** off ephemeral App Service disk into access-tiered Azure Blob containers, separating private student identity documents from public assets and serving them via authenticated proxy or time-boxed signed URLs.
- Migrated **87,440 transcript and certificate records across two institutions** to object storage in a single day's controlled execution, using a mapping-driven, dry-run-by-default migration tool covering 14 mappings across 11 tables.
- Drove adoption of a new storage abstraction across **263 call sites in 91 files** of a 1.17-million-line legacy PHP codebase without an OO refactor, using procedural wrappers to preserve existing call patterns.
- Built a GitHub Actions matrix pipeline deploying a single commit to two independent Azure Web Apps in parallel, replacing manual deployment for a **5,400-file** PHP application.

### High-confidence, non-quantified

- Restored Apache `.htaccess` extensionless routing on nginx by authoring a custom site config and an App Service startup hook that discovers the PHP image's nginx config path at runtime, validates and reloads — avoiding both a mass URL rewrite and ownership of a custom container image.
- Hardened a legacy mysqli/PDO data layer for Azure MySQL with enforced TLS, configurable port, connect timeout and CA path, auto-detecting the Azure host so local development needed no config change.
- Eliminated hardcoded database and payment-gateway credentials from source control, moving all runtime secrets to Azure App Settings.
- Designed an opaque `blob://visibility/path` reference scheme so the database stores no hostnames, endpoints or filesystem layout, and enabled a live incremental migration by making every read path fall back to the legacy filesystem.
- Diagnosed from production data that a bulk migration had silently rewritten ~86,000 business timestamps via `ON UPDATE CURRENT_TIMESTAMP`, establishing the root cause and exact blast radius.

### Per-institution variants — when you need the schools stated separately

Use these when a CV line, portfolio page or application should name one school rather than the combined total. Figures are drawn from §23.2 and §23.3 and are individually defensible. See §23.5 before using them: these are **two deliveries within one role**, not two jobs.

**EKSU-specific:**
- Migrated the EKSU student portal (135,067 student records, 6.30 GB MySQL, 153 tables) from cPanel/Apache to Azure App Service, moving **571,040 file references** to Azure Blob Storage; the instance now serves **14.09M requests/month at a 0.0076% 5xx rate** with 100.00% health-probe availability.
- Carried the September 2026 EKSU student-union election on the migrated platform — **7,188 ballots from 5,730 voters in ~48 hours, peaking at 1,809 ballots/hour** against a 962,046-request day — with no 5xx spike. *(Platform and reliability are mine; the e-voting feature was built by a colleague.)*
- Migrated **86,116 EKSU transcript and certificate records** to object storage in a single controlled execution.

**BOUESTI-specific:**
- Migrated the BOUESTI student portal (33,099 student records, 5.20 GB MySQL, 151 tables) to Azure App Service, moving **104,550 file references** to Azure Blob Storage; the instance serves **8.47M requests/month at a 0.021% 5xx rate** with 99.99% health-probe availability.
- Implemented per-hostname database routing for BOUESTI's **third tenant** (`portalcoe`/`ptpportal` → a separate COE database), the more complex of the two multi-tenant cases.
- Verified migration completeness directly against the BOUESTI production database: **100% of rows touched on migration day were blob-backed, against 0% of all others** — the cleanest available proof that the migration executed as designed.

**Two-delivery framing (one role, both schools named):**
- Delivered an Azure migration and object-storage rollout to **two universities from a single codebase and a single CI matrix** — separate databases, container sets and App Service plans — covering **168,166 student records** and **82,176 distinct monthly users**, at **98.70% file-migration completeness** across both estates.

### Presenting large figures without losing credibility

A real risk with this evidence: the headline numbers are large enough that a recruiter may assume they are invented, especially alongside a two-person team. That reaction is about plausibility, not accuracy. Manage it deliberately.

**The ₦65.51bn payment figure is the main hazard.** Three problems with putting it on a CV: it is a *processed* volume, not revenue you generated or captured; naira figures are hard for UK readers to size; and it invites "if your system handles £X, why are you applying here?" — a question with a good answer that you will not be in the room to give.

**What to do instead, in order of preference:**

1. **Lead with transaction and user counts, not currency.** "1.79M payment transactions" and "82,000 monthly users" are equally impressive, instantly plausible, and carry no valuation baggage.
2. **If you use currency, convert and qualify.** "processed the equivalent of £X in student fee payments" — and always the verb *processed*, never *generated* or *earned*. Do the conversion yourself at a stated rate and date; an unconverted naira figure reads as obfuscation to a UK reader.
3. **Attach the team size as an asset, not an apology.** "Built and operated by a two-engineer team" reframes the numbers from implausible to impressive. The scale and the team size explain each other: this is why one person owns storage, infrastructure, CI/CD and the database layer.
4. **Put the verifiable technical figures first.** 675,590 migrated references, 263 call sites, 22.5M requests, 0.013% 5xx. These are specific, checkable, and unmistakably the output of real engineering.
5. **Keep §23 and the JSON to hand.** Every number here has a stated source, window and calculation. If challenged, you can show the method — which converts a credibility problem into a credibility *advantage*. Most candidates cannot do that.

**On the "why are you job-hunting?" question**, which the figures will provoke: the honest answer is strong. The system processes institutional fee payments; it does not bill per student, for reasons rooted in the institution's history. A fuller answer is PRIVATE and belongs in conversation, not on paper — see §21.

### The institutional-ranking claim — handle with care

You reported that institutional staff, including the vice-chancellor at the time, said verbally that the portal was among the things that lifted EKSU's standing, and that the institution pointed to its graduates — you included — when making its case.

**What I could verify publicly:** EKSU's rankings are real and findable. Webometrics placed it **3rd best state university in Nigeria in 2021**; a later institutional statement cited **2nd best state university** and 14th nationally; and in 2025 the Ekiti State Government described it as the leading state university, citing 12th of 270 Nigerian universities. Sources are listed in §24.

**What cannot be verified, and should not be claimed:** that this system caused or materially contributed to those rankings. The evidence is verbal, second-hand, undocumented, and — critically — unfalsifiable. Ranking methodologies are multi-factor and none isolates the contribution of a student portal.

**Why this matters more than it may seem.** A CV or LinkedIn line reading *"my system made EKSU the best state university in Nigeria"* would be the single most damaging sentence in your entire profile. Any technically literate reader knows a portal cannot be shown to cause a university ranking. The claim is not merely unprovable — it is the kind of claim that makes a reader retrospectively doubt **everything else**, including the 675,590 migrated references and the 22.5M monthly requests, which are solid. You would be trading your most verifiable evidence for your least.

**The defensible version**, which keeps the substance:

> "Built and operated the student records and payments platform for Ekiti State University from 2019 — a period during which the institution was ranked among Nigeria's leading state universities."

That is true, checkable, states the timeframe, and lets the reader draw their own inference. In an interview, if asked, you can add that institutional staff credited the portal as a contributing factor — spoken, attributed, and framed as *their* assessment rather than your claim. That is credible in conversation in a way it never is on paper.

**On the "will UK employers believe any of this?" worry.** Partly, yes — and the answer is not to soften the numbers but to make them checkable. Rankings, institution names and public web presence are verifiable by anyone. Your technical figures come with stated sources and methods. What you should *not* do is attach an unverifiable causal claim to a body of otherwise verifiable evidence, because that is precisely what makes a skeptical reader dismiss the whole.

### On labelling earlier experience "junior"

You asked whether to retro-label pre-2020 work as junior-level. My read:

- **Reasonable, and worth doing.** Undergraduate startup work and self-directed projects are genuinely junior-level, and labelling them so is accurate rather than self-deprecating. It fills timeline, shows progression, and matches how UK employers read a CV.
- **But do not let "junior" bleed into this project.** This is not junior work: seven years of production ownership, two institutions, 22.5M requests/month, a cloud migration and an object-storage rollout. Date it from 2020 and title it to match the responsibility — founding engineer, co-founder, or lead engineer. The contrast between a junior period and this is what makes the progression legible.
- **A caution on the premise.** The pattern you have noticed — that people with a "junior developer" title seem to get UK roles more easily — is more likely about *recognisable employment structure* than the word "junior". What UK screeners look for is a checkable employer, a defined role and a date range. The useful move is to make this project read as structured employment (registered company, named role, clear dates, two named institutional clients) rather than to add a junior title to earlier work and hope it signals legitimacy.
- **Do not omit the earlier period.** Gaps invite more questions than modest early roles do.

### Requires confirmation before use

- Cost, performance or operational-overhead improvement versus the previous hosting — **no baseline data exists**.
- Any specific speedup from the concurrent blob fetch — **implemented, but never measured**.
- Reduction in manual deployment time — **no "before" measurement**.
- Whether I was sole engineer on the Azure migration or part of a team — Git shows me as sole author of every infrastructure file, which is strong but not conclusive.

---

## 17. Portfolio Case Study Material

Raw material for a polished case study; detail lives in the sections cross-referenced.

**Overview.** A legacy PHP/MySQL academic portal serving two Nigerian universities — ~135,000 student records, 1.9M payment transactions, 5.5M result rows — moved from shared cPanel hosting onto Azure, with ~85 GB of sensitive student documents moved from application disk into object storage, while the system stayed live through an admissions cycle, a fee cycle and a student-union election.

**Challenge.** ~1.17M lines of procedural PHP across 5,427 files, 5,039 raw `mysqli_query()` call sites, no test suite, assuming Apache rewriting, local filesystem writes and MariaDB dialect tolerance. The target platform provides none of those. The constraint that shaped every decision: it could not go down, and it could not be rewritten.

**My role.** Cloud migration and storage platform engineer — not the application's author (121 of 155 commits belong to a colleague). I own the Azure runtime, the deployment pipeline, the database layer's Azure capabilities, and the entire storage abstraction and migration. Full breakdown: §4.

**Solution and architecture.** See §5 (architecture diagram) and §4 (components). Three pieces: runtime compatibility (nginx config + startup hook), storage (dependency-free Blob client, three-tier visibility, opaque `blob://` references, authenticated proxy, concurrent materialisation, permanent local fallback), and delivery (GitHub Actions matrix, secrets in App Settings).

**Technologies.** §7.

**Interesting engineering problems.** §10 and §12 in full. The ones to lead with: reproducing Apache semantics on a platform with no config hook; implementing two unforgiving Azure signing schemes by hand; designing a migration whose fallback invariant lets it run incrementally against a live database; and discovering, two months later and from production data, that my own migration had silently destroyed ~86,000 business timestamps (§14).

**Two deliveries, one platform.** The case study can be written as one project or split into two institutional case studies — §23 holds the per-school figures and the rules for each framing. The honest spine either way: the storage layer, migration tool, routing fix and pipeline were built once; the migration and cutover were executed twice, against separate databases, separate container sets and separate user bases, with separately measured outcomes (EKSU 14.09M requests / 0.0076% 5xx / 571,040 references; BOUESTI 8.47M / 0.021% / 104,550). BOUESTI is the more interesting routing case — it carries a third tenant via per-hostname database routing that EKSU has no equivalent of.

**Measurable evidence.** §8 in full, split per school in §23. Headlines: 22.56M requests / 31 days at 0.013% 5xx across two deployments; 100.00% and 99.99% health-probe availability; 675,590 migrated references at 98.70% completeness; ~1.2M blobs / ~85.8 GB; 263 call sites across 91 files; 87,440 records migrated in one day's execution; 82,176 distinct users in 30 days.

**Lessons.** §14.

**What I'd improve next, in priority order.** (1) Shared session store — Redis or a DB handler — because local file sessions are the single thing blocking horizontal scaling, and the first attempt was reverted. (2) Managed Identity + RBAC to retire the long-lived storage account key. (3) A test suite for the signing code. (4) Finish the `ebooks` migration and add a completion tracker so no mapping can stall behind the fallback. (5) Enable MySQL HA and a second App Service worker. (6) Rewrite repository history to shed the 261 GB `.git`.

## 18. Interview Story Bank

**S1 — Production debugging / owning a mistake (strongest story).**
*Situation:* auditing production data two months after a storage migration I had run. *Task:* establish real usage volumes for transcript and certificate requests. *Action:* the yearly breakdown showed 41,835 transcript requests in 2026 against ~1,500–5,000 in prior years — implausible. Monthly breakdown put 39,035 of them in August alone; daily put 37,346 on 15 August, the exact date of my migration execution. I checked `information_schema` and found `appliedDate` declared `ON UPDATE CURRENT_TIMESTAMP`, then ran a correlation: of the 37,346 rows stamped that day, 37,346 (100%) held blob references, against 2,926 of 15,111 for every other date. *Result:* confirmed my migration's `UPDATE` statements had silently rewritten the application date on ~86,000 records. I reported it rather than quietly correcting the statistic, and the general rule I now apply is to enumerate `ON UPDATE CURRENT_TIMESTAMP` columns and triggers before any bulk update on a legacy schema. *Evidence:* schema definition, correlation query, commit date.

**S2 — Architecture / making an irreversible change reversible.**
*Situation:* 571,000+ database rows referencing files on disk, on a live system with no tests. *Task:* move them to object storage without downtime. *Action:* instead of a cutover, I chose an opaque `blob://` reference and made the invariant *every read path falls back to the legacy file if the value isn't a blob reference* — preview, download, ZIP, PDF, delete. That let me migrate table by table, column by column, with a dry-run-by-default tool and `--only=` batching. *Result:* 571,040 references migrated, zero legacy remaining in those columns, no downtime. *Reflection:* the fallback also removed the pressure to finish — `ebooks` is still unmigrated two months on. I'd pair the fallback with an explicit completion tracker next time.

**S3 — Platform constraints / knowing when not to rewrite.**
*Situation:* after lift-and-shift, every internal link 404'd because nginx ignores `.htaccess`. *Task:* restore extensionless routing. *Action:* I rejected rewriting URLs across 5,427 untested files, and rejected a custom container image because the client wanted managed hosting, not PHP patching. Instead: an nginx config reproducing the rewrite, installed by a startup script that probes four candidate config paths, runs `nginx -t` before reloading, and logs to `/home/LogFiles`. *Result:* live on both apps. *Trade-off I'd state honestly:* it's coupled to App Service's internal layout and could break on an image update — which is exactly why it validates and logs.

**S4 — Performance / understanding a changed cost model.**
*Situation:* pages that ZIP several hundred student documents started timing out after the storage cutover. *Task:* restore them. *Action:* diagnosed that each `fopen()` had become an HTTPS round trip plus a TLS handshake, so an N-file export scaled linearly into the FastCGI timeout. Built `materializeMany()` on `curl_multi` with a bounded in-flight window, streaming to disk, isolating per-item failures and caching within the request. The non-obvious part was the scheduling loop: a job failing before reaching curl consumes its slot without entering the active set, so refill must happen at the top of the iteration — otherwise a batch of such failures spins with nothing in flight. *Result:* implemented and in production. *Honest caveat:* I never captured before/after timings, which I regret and would fix.

**S5 — Technical judgement / dependency choice.**
Why I implemented Azure's SharedKey and SAS signing by hand rather than using the SDK: the PHP SDK is effectively unmaintained and would have pulled a dependency tree into a PHP 8.3 app still carrying PHP-7-era assumptions. The cost is that I own correctness of two unforgiving signing schemes, with API versions pinned in source and no test coverage — which is the single piece of this system I'd write tests for first.

**S6 — Reliability under a known deadline.**
The September 2026 student-union election ran on this platform: 7,188 ballots from 5,730 voters over ~48 hours, peaking at 1,809 ballots in one hour, against 962,046 requests on the busiest day. Response time rose to ~2 s and 5xx did not spike. I'd be explicit that I built the platform and a colleague built the voting feature — and that the 100% health-probe availability came with no HA and a single worker, so the margin was thinner than the numbers suggest.

**S8 — Multi-tenancy without a tenancy model.**
*Situation:* one codebase had to serve two universities with completely separate data, documents and users. *Task:* isolate them without a tenant column, which the application has never had. *Action:* isolation is achieved at deployment time rather than in code — the same commit deploys to two App Service instances via a CI matrix, each pointed at its own database, its own three blob containers (`eksu-portal-*` / `bouesti-portal-*`) and its own App Settings. BOUESTI additionally carries a third tenant through per-hostname routing in `server.php` to a separate `coeikere-db`. *Result:* both run in production off one commit stream — 14.09M and 8.47M requests in 31 days, with the storage migration executing independently against each (571,040 and 104,550 references). *What I'd say about the trade-off:* tenant-specific behaviour now lives in environment variables rather than code, which is invisible to anyone reading the repository, and a bad commit reaches both universities simultaneously because there is no per-school release isolation. I'd argue that was the right call for two tenants and the wrong one for ten.

**S9 — Proving a migration worked, twice.**
*Situation:* I needed to show the blob migration had actually completed, not just that the script exited cleanly. *Action:* queried both production databases for the ratio of `blob://` values to surviving legacy paths, per column. *Result:* every high-volume document mapping is fully migrated on both instances; 98.70% of all mapped values overall. The BOUESTI data gave the cleaner proof of *when* it ran — 1,324 rows stamped 2026-08-15, 100% of them blob-backed, versus 0% of all other rows, with no pre-existing blob rows to muddy the signal. *Why this is a good story:* it shows verification against production rather than trust in a script, and it is the same query that surfaced the timestamp defect in S1.

**S7 — A decision I can no longer defend.**
I added Redis-backed sessions and removed them the next day; I added a staging deployment slot and removed it eight weeks later. Neither commit message records why. The session decision still constrains the system — local file sessions are why both apps run a single worker. The lesson is cheap and concrete: a two-line "why" in a revert commit, or an ADR, would have preserved reasoning that is now gone.

---

## 19. Technical Article / Blog Opportunities

**A1 — "Your bulk UPDATE is rewriting columns you didn't name."**
*Problem:* `ON UPDATE CURRENT_TIMESTAMP` on a legacy schema turns a one-column migration into a silent rewrite of business dates. *Why interesting:* it is a well-documented MySQL feature that almost nobody accounts for in migration review, and the damage is invisible until someone reads the data months later. *What I'd explain:* the mechanism, how to enumerate affected columns from `information_schema`, how to pin them in the `SET` clause, and how to detect the damage after the fact via correlation. *Anonymise:* institution names, table and column names, exact row counts. *Do not publish:* the schema, the migration script, or anything identifying the universities.

**A2 — "Running an Apache-era PHP app on Azure App Service nginx."**
*Problem:* `.htaccess` is ignored and the platform gives you no supported config hook. *Why interesting:* a very common lift-and-shift wall with little good public writing. *What I'd explain:* the `try_files` → named-location equivalent of the Apache rewrite, why the startup hook must probe multiple config paths, and why `nginx -t` before reload is non-negotiable. *Publishable:* the config pattern is generic. *Do not publish:* hostnames, FastCGI timeout values tied to this app's report pages, or the deny rules revealing internal file layout.

**A3 — "Signing Azure Blob requests by hand, and when that's the right call."**
*Problem:* SharedKey's canonical string-to-sign and service SAS are strict, differently-shaped, and fail identically with an opaque 403. *Why interesting:* a concrete walkthrough with the gotchas (header canonicalisation, encoding the path but not the separators, `Content-Length` suppressed for GET/DELETE, API-version pinning). *What I'd share:* the decision framework for hand-rolling versus an SDK, and the honest conclusion that if you hand-roll it, you owe it tests. *Do not publish:* account names, container names, TTL values, or any real signature.

**A4 — "Migrate a live system by making the old path a permanent fallback."**
*Problem:* moving hundreds of thousands of file references on a system that cannot go down. *Why interesting:* the fallback invariant is the whole design, and the article can be honest that it has a cost — one mapping is still unmigrated because nothing breaks. *What I'd share:* the opaque-reference scheme, dry-run-by-default tooling, and the argument for a completion tracker. *Anonymise:* all domain specifics.

**A5 — "Object storage changes your performance model, not just your storage."**
*Problem:* `fopen()` → HTTPS round trip; linear loops become timeouts. *What I'd explain:* how to find these loops before they find you, and a `curl_multi` bounded-concurrency pattern in plain PHP including the slot-refill bug. *Caveat I'd state in the piece:* I did not capture before/after numbers, so the article teaches the pattern rather than claiming a speedup.

---

## 20. LinkedIn / Social Content Opportunities

Each is grounded in a specific verified finding, not a generic reflection.

1. **"My migration deleted 86,000 dates and I found out two months later."** The `ON UPDATE CURRENT_TIMESTAMP` story (§14). Credible because it is a mistake, found by me, with the diagnostic chain shown. The strongest and most honest post available here.
2. **"The hardest part of our cloud migration wasn't the database."** `.htaccess` vs nginx, TLS enforcement, `CAST(x AS INT)`, PHP 8 throwing where PHP 7 returned `false`, sessions on a CIFS mount. Evidence: the recovered modernization log and the nginx/startup work.
3. **"571,000 file references, zero downtime, one invariant."** The fallback-path design — and the honest coda that the same invariant is why one mapping is still unmigrated.
4. **"Two reverts, no reasons: a short case for writing down why."** Redis sessions and the staging slot. Specific, self-critical, and lands a practice everyone recognises.
5. **"We ran a student election on a single App Service worker."** 7,188 ballots, 5,730 voters, 1,809 in the peak hour, 962k requests that day, no 5xx spike — with the caveat that no HA made the margin thinner than it looks.
6. **"When is hand-rolling a cloud SDK the right answer?"** The SharedKey/SAS decision, argued both ways, ending on the test-coverage debt.
7. **"263 call sites, one procedural wrapper."** How to introduce a modern abstraction into 1.17M lines of procedural PHP without an OO refactor.

---

## 21. Private vs Public-Safe Information

### PRIVATE — do not publish, highest sensitivity

**The predecessor fraud incident must never be published in any form.** It concerns a real, identifiable individual who was reportedly arrested, and a named institution. Publishing it — on LinkedIn, in an article, in a portfolio case study, or in a public CV — risks defamation exposure and reflects poorly on you regardless of accuracy. This holds even with names removed: the combination of institution, role and timeframe would identify the person to anyone local.

In an interview, if context is genuinely needed, the safe formulation is: *"we replaced a system that had been discontinued after a loss of institutional trust"* — no detail, no individual, no amount, and only if asked. Do not volunteer it.

**Also private:** your compensation history and current arrangement; the commercial terms with either institution and any future per-student billing discussion; the reasons the current arrangement is below market. These are legitimate inputs to your own negotiation and salary decisions, and they are recorded here for that purpose only.

### PRIVATE — standard

Azure subscription and resource IDs, resource group name, Web App names, storage account name, container names, MySQL server hostname and database names; all credentials, keys, publish profiles and connection strings; the specific list of App Settings names (it maps the system's attack surface); institution identification tied to specific production statistics; per-table row counts, the ₦48.78bn payment figure and the 1.55M transaction count; the `blob://` scheme details and the storage proxy's reference encoding; the rate-limit thresholds and allow-list paths in `server.php`; the hand-rolled signing implementation and its pinned API versions; the identified data-integrity defect tied to a named institution; CSRF scope configuration and feature-flag names; everything in `activity_log` and every table containing student or staff records.

### PUBLIC-SAFE — with the stated anonymisation

- The architecture *pattern*: legacy PHP on App Service, nginx rewrite restoration, blob-backed storage with three visibility tiers, matrix CI/CD to two deployment targets — described generically, without names.
- The `ON UPDATE CURRENT_TIMESTAMP` lesson, with "a legacy MySQL schema" and "~86,000 records" rather than institution, table or column names.
- The fallback-invariant migration pattern, described generically.
- The `curl_multi` bounded-concurrency pattern, as generic PHP.
- The SharedKey/SAS vs SDK decision framework, without any real values.
- Order-of-magnitude scale only: "a portal serving 100,000+ student records", "20M+ requests/month across two deployments", "500,000+ file references migrated", "~85 GB of documents". **Never** the exact figures, and never alongside the institution names.
- My role framing: "cloud migration and storage platform engineering on an existing large legacy PHP system" — which is both accurate and the honest boundary.

**When in doubt:** anything that pairs an institution name with a production number is private.

---

## 22. Evidence Gaps and Further Metrics Worth Collecting

| Gap | Why it matters | Where it might exist | Question to ask |
|---|---|---|---|
| Why Redis sessions were reverted | It's the current scaling blocker and the decision is unrecoverable from Git | Team chat / PR discussion around 16–17 Aug 2026; the reverted branch `codex/redis-session` | "What broke when we tried Predis sessions on App Service?" |
| Why the staging slot was removed | A release-engineering decision I made and currently cannot defend | Chat / notes around 10 Oct 2026 | "Was the slot removed for cost, swap failures, or deploy time?" |
| Before/after bulk-export timings | Would convert C12 from "implemented" to a quantified performance win | App Service logs before 10 Aug 2026; the SIGKILL-after-1410s entry in the modernization log suggests a pre-existing baseline | "Do we have request-duration logs for the ZIP/PDF export pages before and after the storage change?" |
| Previous cPanel hosting cost and spec | The only route to a credible cost/efficiency claim | Client billing records; the hosting invoice | "What was the previous hosting costing per month, and what were the specs?" |
| Exact production go-live date **on Azure** | Dates the cloud migration precisely; Azure metrics retention only reaches 12 Sep. *(Note: the system's overall production start is now resolved — EKSU 2020, BOUESTI 2021 — see §4.)* | DNS cutover records; App Service activity log (90-day retention — check soon) | "When did the DNS for the portal domains move to the Azure apps?" |
| Retention and retrievability of the off-cloud EKSU `activity_log` | The pre-Aug-2026 audit trail was deliberately left on the old host (§11). Its usefulness depends entirely on that copy still existing and being restorable — which is an operational risk, not a migration defect | The old cPanel host; any archive of it | "Where exactly is the historical EKSU `activity_log` now, who can restore it, and how long will that host remain available?" |
| Dated artefacts from Sept 2019 – mid 2020 | Would convert the September 2019 start from user-reported to verified. The production database cannot supply it: that phase used session-tagged tables with no row timestamps, and payments were not integrated until 2020 | Early institutional correspondence; the original repository; the first deployment or hosting account records | "Do we have anything dated from the 2019 build-and-load phase — an email, an invoice, a hosting signup?" |
| Dated proof of the founding timeline | Would convert C18 from STRONGLY SUPPORTED to VERIFIED and is the single highest-value gap remaining | The original 2019–2022 repository or backups; the first institutional agreement or NYSC posting letter; early email with the university | "Can we recover the pre-2023 repository, and do we still have the original engagement letter?" |
| Pre-migration traffic baseline | Would let me claim capacity improvement rather than just capacity | cPanel AWStats/Webalizer on the old host, if still accessible | "Do we still have access to the old server's traffic stats?" |
| Number of staff/administrators using the portal | `activity_log` shows only 203 staff in session 2026, which likely undercounts | The institutions' registry/ICT units | "How many staff accounts are actually in use across the registry and faculties?" |
| Whether I was sole engineer on the Azure migration | Git strongly implies it; confirmation makes C1 unimpeachable | The team | "Was anyone else working on the Azure infrastructure alongside me?" |
| Whether the timestamp loss caused operational harm | Determines whether S1 is a lesson or an incident | The registry / transcript office | "Does anyone rely on `appliedDate` for transcript SLA reporting?" |
| Restore test for the 7-day MySQL backup | Backups that have never been restored are not backups | Azure portal / ops records | "Have we ever test-restored the Flexible Server backup?" |
| Why BOUESTI averages ~2.6× EKSU's response time | 0.496 s vs 0.19 s on the same plan size with lower traffic; unexplained, and it would be a weak spot if raised in an interview | Application Insights dependency/request breakdown on `bouesti-app-prod`; the COE routing path is a candidate | "Is the slower BOUESTI response time the COE host routing, a query difference, or just the heavier activity_log?" |
| Per-school split of blob storage capacity | The ~85.8 GB / ~1.2M blob figures are account-wide and cannot be attributed per institution from metrics alone | Per-container blob enumeration, or Storage Insights with a container dimension | "Can we get per-container capacity broken out for the eksu-* and bouesti-* containers?" |
| Why `activity_log.session` on BOUESTI still reads 2025 | Current events are tagged to the prior session, which would corrupt any session-based reporting on that instance | The session rollover procedure / `setup` table | "Has the 2026 session been activated on BOUESTI, and what sets `activity_log.session`?" |

---

## 23. Per-Institution Evidence Split

This section exists so the evidence can be presented **either** as one engagement **or** as two institutional deliveries, without restating anything inaccurately. Use it as the lookup when you write a CV line, a portfolio page or an interview answer and need to know what belongs to which school.

### 23.1 What is genuinely shared versus genuinely separate

| Layer | Shared across both | Separate per school |
|---|---|---|
| Codebase | One repository, one commit history | — |
| Storage abstraction (`Storage.php`), migration tool, nginx config, startup hook, health endpoint | Written once, serves both | — |
| CI/CD pipeline | One workflow, matrix over two targets | Separate publish profiles |
| Azure App Service | — | `eksu-app-prod` / `bouesti-app-prod`, separate plans |
| Database | Same Flexible Server instance | Separate databases: `eksu-db` / `bouesti-db` (+ `coeikere-db`) |
| Blob storage | Same storage account | Separate containers: `eksu-portal-*` / `bouesti-portal-*` |
| App Settings | — | 52 (EKSU) / 32 (BOUESTI), separately scoped |
| Hostnames / user base / data | — | Entirely separate |

**The honest line:** the *engineering* was built once; the *delivery, migration and operation* happened twice, against two separate production estates with separate data, separate risk and separately measured outcomes.

### 23.2 EKSU — standalone evidence

| Measure | Value | Window |
|---|---|---|
| HTTP requests | 14,092,342 | 31 days to 13 Oct 2026 |
| HTTP 5xx / rate | 1,064 / 0.0076% | same |
| Peak single day | 1,239,422 requests | 12 Oct 2026 |
| Health-probe availability | 100.00% | 7 days to 13 Oct 2026 |
| Mean response time | 0.19 s (median of dailies) | 14 days to 12 Oct 2026 |
| Database | 6.30 GB, 153 tables | 13 Oct 2026 |
| Student records | 135,067 | — |
| Staff records | 2,900 across 193 department codes | — |
| Blob references migrated | 571,040 | — |
| Legacy values remaining | 8,386 | — |
| Distinct users, 30 days | 53,143 | 14 Sep – 14 Oct 2026 |
| Peak daily active users | 19,769 | 12 Oct 2026 |
| Successful payments | 1,551,157 (83.55%), ≈₦48.78bn, 223,663 payers | full history |
| Raw score rows | 5,517,117 | — |
| CBT | 3,472 tests, 127,083 questions, 374,799 results | — |

**EKSU-only distinctives:** **1,822,938 historical academic records migrated in** (raw scores, level records and student records tagged to pre-2019 sessions, with session configuration back to 1990) — BOUESTI has effectively none, having started fresh. The student-union election (7,188 ballots, 5,730 voters, 1,809 peak hour, Sept 2026) ran here and **only** here. The heavy transcript/certificate workflow (120,744 combined requests) is EKSU's, as is the bulk of the migration-day timestamp damage (86,116 of 87,440 rows).

### 23.3 BOUESTI — standalone evidence

| Measure | Value | Window |
|---|---|---|
| HTTP requests | 8,469,006 | 31 days to 13 Oct 2026 |
| HTTP 5xx / rate | 1,776 / 0.021% | same |
| Health-probe availability | 99.99% | 7 days to 14 Oct 2026 |
| Mean response time | 0.496 s | 7 days to 14 Oct 2026 |
| Database | 5.20 GB, 151 tables | 14 Oct 2026 |
| Student records | 33,099 | — |
| Staff records | 747 | — |
| Blob references migrated | 104,550 | — |
| Legacy values remaining | 500 | — |
| Distinct users, 30 days | 29,033 | 14 Sep – 14 Oct 2026 |
| Distinct users, 90 days | 39,096 | 16 Jul – 14 Oct 2026 |
| Peak daily active users | 16,075 | 24 Aug 2026 |
| Successful payments | 238,827 (87.98%), ≈₦16.73bn, 61,758 payers | full history |
| Raw score rows | 1,335,391 | — |
| Activity-log rows | 8,471,215 | — |

**BOUESTI-only distinctives:** it carries a **third tenant** — `server.php` routes the `portalcoe`/`ptpportal` hostnames to a separate `coeikere-db`, which EKSU has no equivalent of, making BOUESTI the more complex routing case. It also has the **higher payment success rate** (87.98% vs 83.55%) and the **cleaner migration proof** (100% vs 0% correlation, because it had no pre-existing blob rows). No e-voting in production.

### 23.3b College of Education, Ikere-Ekiti — the third deployment

Served by its own database (`coeikere-db`), still reachable in production via the `portalcoe`/`ptpportal` host routing in `server.php`. This institution became BOUESTI; the two instances ran in parallel through the transition.

| Measure | Value |
|---|---|
| Database | 0.25 GB, 151 tables |
| Students | 11,785 (entry years from 2012 — legacy import) |
| Staff | 278 |
| Raw scores | 132,243 |
| Payment transactions | 41,707 |
| Activity-log events | 498,788 |
| First logged activity | 11 Dec 2020 |
| Peak year | 2022 — 206,754 events, 14,384 distinct users |
| Wind-down 2023 → 2026 | 118,333 → 52,973 → 22,340 → 1,457 events |

**How to use this.** It supports a distinct and unusual claim: **three institutional deployments, and a parallel-run migration across an institution's change of status.** Do not fold its figures into the EKSU/BOUESTI combined totals in §23.4 — those were computed before this instance was identified and deliberately exclude it. Quote it separately.

### 23.4 Combined figures — use only when presenting as one engagement

| Measure | Combined | Composition |
|---|---|---|
| HTTP requests, 31 days | 22,561,348 | 14,092,342 + 8,469,006 |
| HTTP 5xx rate | 0.013% | 2,840 of 22,561,348 |
| Blob references migrated | 675,590 | 571,040 + 104,550 |
| Migration completeness | 98.70% | 675,590 of 684,476 |
| Records migrated 15 Aug 2026 | 87,440 | 86,116 + 1,324 |
| Database footprint | 11.50 GB across 2 databases | 6.30 + 5.20 |
| Student records | 168,166 | 135,067 + 33,099 |
| Distinct users, 30 days | 82,176 | 53,143 + 29,033 |
| Successful payments | 1,789,984, ≈₦65.51bn | 1,551,157 + 238,827 |
| Distinct payers | 285,421 | 223,663 + 61,758 |

**Do not sum table counts.** EKSU has 153 tables and BOUESTI 151, but these are the *same schema* deployed twice, not 304 distinct table designs. Summing them would overstate the data model by 2×. The same logic applies to anything describing the codebase (5,427 PHP files, 263 call sites, 151-table schema): it was built once and deployed twice — never double it.

**Two caveats on every combined figure.** (a) Blob storage totals (~1,197,452 blobs / ~85.77 GB) are **account-wide and cannot be split** — quote them only as combined, never as one school's. (b) Combined **distinct** user and payer counts (82,176 and 285,421) are the weakest sums here. Adding two distinct-counts yields a true distinct total only if the two populations are disjoint. Institution-specific identifier formats make overlap very unlikely, but I did not cross-match identifiers to prove it — doing so would mean handling personal data for no analytical gain. Treat these two as upper bounds, not exact counts.

### 23.5 How to present this on a CV — one role or two?

**Present as one role, two deliveries. Do not present as two jobs.** If this was a single engagement or employer, listing EKSU and BOUESTI as separate work experiences would overstate your history — an interviewer who finds one repository and one commit stream will read it as padding, and it would undercut otherwise strong, verifiable evidence. The scale is real; the chronology is one period.

What is fully defensible:

- **One role line, combined scale** — "migrated two university portals … 675,000+ file references, 22.5M requests/month". Strongest headline; use §23.4.
- **One role line, two named deliveries** — "delivered to two institutions (135k and 33k student records) from a single codebase and pipeline". Shows multi-tenant delivery without inflating history.
- **Separate portfolio case studies** — entirely legitimate. Two pages, each using §23.2 or §23.3 figures only, cross-referencing the shared platform work.
- **Interview answers** — pick the instance that fits the question. Election capacity → EKSU. Multi-tenant routing complexity → BOUESTI. Migration method → either, with the other as replication evidence.

**Never do this:** quote a combined figure under one school's name; quote EKSU's 100% availability or its election for BOUESTI; or claim "zero legacy remaining" for either (it is 98.70% combined, with 8,386 and 500 outstanding respectively).

---

## 24. Appendix — Evidence Notes

**Method.** Repository and Git history read directly. Azure state read via `az` CLI (read-only queries: `webapp show`, `appsettings list` restricted to names, `mysql flexible-server show`, `storage account list`, `monitor metrics list`). Production MySQL accessed via the `mysql` client with `--init-command="SET SESSION TRANSACTION READ ONLY"` against `eksu-db`; only `SELECT`, `COUNT`, `SUM`, `GROUP BY` and `information_schema` reads were issued. No write, DDL, migration or administrative statement was executed at any point.

**Attribution method.** Git `--author` filtering and per-file `git log` were used to establish ownership. Files where every commit is mine are treated as sole-authored; files where my commits add a specific capability to a colleague's file are described that way explicitly.

**On the diffstat.** My 33 commits show ~40,596 insertions / ~38,947 deletions across 1,409 file-paths. These figures are inflated: commit `62b5805f` vendored Predis (hundreds of files) and `3008db2a` reverted it the next day, and `cbf3b356` removed 85 committed upload files. Line counts are a poor proxy for contribution here; the file-level ownership in §4 is the reliable evidence.

**Recovered document.** `MODERNIZATION_PROGRESS.md` (470 lines) was deleted from the working tree on 20 Aug 2026 but recovered from Git (`git show 5c4f5274^:MODERNIZATION_PROGRESS.md`). It is the richest narrative source in the project. Its session entries are signed "Claude", indicating AI-assisted sessions; commit authorship was used to attribute the work they describe.

**Azure Monitor retention.** Request and status-code metrics at 1-day grain returned 31 data points from a 66-day query window, so the effective window is 12 Sep – 13 Oct 2026. Longer-horizon traffic claims are not available from this source.

**Known measurement caveats, restated.** `information_schema.table_rows` is an InnoDB estimate and was used only for the size overview; all headline counts use exact `COUNT(*)`. `paymentAmount` is a `varchar` and non-`Successful` rows contain non-amount values — payment totals are restricted to `paymentStatus='Successful'`. `transcript`/`transc_certificate` `appliedDate` values dated 2026-08-15 are migration artefacts, not application dates. "Distinct users" counts distinct `activityUser` identifiers with at least one logged event.

**Public sources consulted (for the institutional-ranking context in C22 only).** EKSU ranking positions were checked against public reporting rather than taken on report: [Ekiti State University — Wikipedia](https://en.wikipedia.org/wiki/Ekiti_State_University); [EKSU ranked 3rd best state university (institutional statement)](https://eksu.edu.ng/eksu-one-of-the-best-sought-institutions-ranked-as-3rd-best-state-university/); [Ekiti State Government](https://ekitistate.gov.ng/?p=30278); [Times Higher Education — Ekiti State University](https://timeshighereducation.com/world-university-rankings/ekiti-state-university). Sources disagree on the exact position by body and year, which is why C22 cites a range rather than a single figure. **No source links any ranking to this system**, and none was sought to, since the causal question is not resolvable from public data.

**PII handling.** Student and staff identifiers, email addresses and user-agent strings were encountered during schema inspection and are deliberately excluded from all outputs. Every figure in this dossier is an aggregate.

**Scope.** This dossier covers **both** production instances. EKSU's database was queried on 13 Oct 2026 and BOUESTI's on 14 Oct 2026, both read-only. BOUESTI's credentials were read from the `bouesti-app-prod` App Service settings at the user's instruction; they resolve to the same server-administrator account on the same Flexible Server, so no additional credential was in fact involved.

**Correction log.** The 13 Oct draft stated "zero legacy paths remaining in the migrated columns". That overstated completeness: the legacy side of `leave_of_absence.attachedFile` had not been queried, and it holds 5,906 legacy values on EKSU and 352 on BOUESTI. §8.2 now reports 98.70% completeness with the four unmigrated mappings itemised, and claim C4 and the corresponding CV bullet have been restated. No other figure changed.

---

*Compiled 13 October 2026 from read-only inspection of source, Git history, Azure Resource Manager, Azure Monitor and the EKSU production database.*
