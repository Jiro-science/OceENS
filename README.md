# OcéEns II

Course evaluation platform built for the EPF engineering school.

## Overview

**OcéEns II** lets programme managers, facilitators, campus managers and administrators create and manage evaluation surveys (*sondages*) for EPF's programmes, and lets students answer them. Answers can be exported, visualised, and summarised by an LLM (*synthèses*). The interface uses EPF's official visual identity and is in French; the vocabulary boundary between this documentation and the product is described in [`CONTEXT.md`](CONTEXT.md).

### Tech stack

| Component | Technology |
|-----------|------------|
| **Framework** | FastAPI (Python 3.12) |
| **Authentication** | Microsoft Entra ID (Azure AD) via OAuth 2.0 / MSAL and Microsoft Graph, or the [development login](#development-login) |
| **Database** | SQLite (via SQLAlchemy + SQLModel) |
| **Templating** | Jinja2 (server-side rendering) |
| **Frontend** | HTML / CSS / JavaScript, no framework |
| **Server** | Uvicorn |
| **Logging** | Python's standard `logging` module, through the Uvicorn handlers |
| **Exports** | Pandas (CSV) |
| **Free-text summaries** | Separate daemon calling an LLM (`requests-cache`, `markdown-it-py`) |

---

## Getting started

These steps take a fresh clone to a running application in `dev` mode, with no Entra credentials and no LLM key.

### Prerequisites

- Git
- Python 3.12, for a local run
- A running Docker daemon (Docker Desktop on Windows and macOS), for a Docker run

### 1. Clone and create `.env`

```bash
git clone https://github.com/EPF-MDE/OceENS.git
cd OceENS
cp .env.example .env            # Windows PowerShell: Copy-Item .env.example .env
```

`.env.example` ships `AUTH_MODE=dev`, which uses the [development login](#development-login) instead of Microsoft Entra ID: no Entra credentials are needed, and no other value has to be filled in. Every variable is described in the [configuration reference](#configuration-reference).

> [!CAUTION]
> `AUTH_MODE=dev` lets anyone log in as anyone. Never deploy with it; see the [deployment checklist](#deployment-checklist).

### 2a. Run locally with Uvicorn

**Windows (PowerShell)**

```powershell
py -3.12 -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\uvicorn.exe main:app --reload --port 8000
```

**macOS / Linux (bash)**

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/uvicorn main:app --reload --port 8000
```

The commands call the virtual environment's executables by their path, so the environment does not need to be activated (on Windows, PowerShell's default execution policy blocks `Activate.ps1`). `--reload` restarts the server when a source file changes.

### 2b. Run with Docker Compose

```bash
docker compose up --build
```

`docker compose` refuses to start without a `.env` file (`env file .env not found`): step 1 is required. The image runs Uvicorn **without** `--reload`: after a code change, run `docker compose up --build` again. The container mounts two host folders: the database folder (`./database/` by default, see `LOCAL_DATABASE_DIR`) and `./import/`, which holds the seed CSV files.

### 3. Log in

Open **http://localhost:8000**. On first startup the application creates the SQLite database (`database/db_oceens.db`) and fills it with a demo data set. In `dev` mode, `/login` leads to `/dev/login`, which lists the database's users by role: click one to log in as that user. `antoine.gademer@epf.fr` is an admin in the demo data.

The application runs without an LLM key: everything works except free-text summaries, which need a key and the summaries daemon (see [LLM providers](#llm-providers-free-text-summaries)).

To check that a change did not break startup, configuration or the container, follow the [manual smoke test](docs/smoke-test.md).

---

## Configuration reference

The application reads its configuration from environment variables. `main.py`, `core/auth.py` and `summaries_generator_daemon.py` call `load_dotenv()`, so a `.env` file at the project root is enough; variables already set in the environment take precedence over `.env`. [`.env.example`](.env.example) lists every variable below and is the template to copy.

| Variable | Default | Description |
|----------|---------|-------------|
| `AUTH_MODE` | `entra` | `entra` (Microsoft Entra ID) or `dev` ([development login](#development-login)), case- and whitespace-insensitive. Any other value stops the application at startup (exit code 1). |
| `DEV_LOGIN_KEY` | unset | `dev` only. When set, every development login must provide it (`key` field), otherwise `401`; when unset, the login is open. Ignored, with a warning, in `entra`. |
| `SECRET_KEY` | unset | Signs the session cookies: anyone who knows it can forge a session for any user, including an admin. **Required in `entra`**: when missing or empty, the application logs a critical error and exits with code 1. Optional in `dev`: when missing, a random key is drawn at each start (with a warning) and sessions are lost on restart. A known key (shared, copied from an example…) lets anyone forge a cookie and bypass `DEV_LOGIN_KEY` in `dev` too. Generate one with `python -c "import secrets; print(secrets.token_urlsafe(32))"`. |
| `ALLOWED_DOMAINS` | `epf.fr,epfedu.fr`, except for `entra` logins | Comma-separated e-mail domains allowed to log in (`403` otherwise) and to be added as users or enrolled as students. In `entra`, leaving it unset rejects every login. |
| `ENTRA_CLIENT_ID` | unset | Entra application (client) ID. Required in `entra`: the application exits with code 1 if it is unset (an empty value is not caught at startup and fails at login). Not needed in `dev`. |
| `ENTRA_CLIENT_SECRET` | unset | Entra client secret. Required in `entra`, like `ENTRA_CLIENT_ID`. |
| `ENTRA_TENANT_ID` | unset | Entra tenant ID. Required in `entra`, like `ENTRA_CLIENT_ID`. |
| `REDIRECT_URI` | `https://localhost/auth/callback` | URL Microsoft redirects to after login; must match the one registered in Entra. `entra` only. |
| `LOCAL_DATABASE_DIR` | `database/` at the project root | Folder of the SQLite database `db_oceens.db`, created if missing; a relative path is resolved from the project root. With Docker Compose, it is the host folder mounted as the container's `/app/database`. |
| `LLM_API_KEY` | unset | API key of the default LLM provider (Ollama EPF). When empty, the application starts normally and requested summaries are marked as a configuration error. |
| `RUN_SUMMARIES_DAEMON` | unset | `1`, `true`, `yes` or `on`: the application starts `summaries_generator_daemon.py` as a child process at startup and stops it on shutdown. Leave unset when the daemon is started separately (`launch.sh` does). |
| `LLM_*`, `*_API_KEY` | – | API keys of additional LLM providers: a provider stores the *name* of its key's variable, never its value (see [LLM providers](#llm-providers-free-text-summaries)). |

> [!CAUTION]
> Never commit `.env`. It is listed in `.gitignore`, like `*.db` files (`database/db_oceens.db`, `cache_llm.db`).

---

## Roles

- `student`: answers the surveys they are enrolled in.
- `program_manager:<code>`: manages the surveys of their programme(s).
- `facilitator:<code>`: runs the surveys of their programme(s).
- `campus_manager:<campus>`: campus-wide scope.
- `admin`: general administration.

A user can hold several roles, each with its own scope (programme codes or campuses separated by `;`).

---

## Main pages and routes

| Route | Description |
|-------|-------------|
| `/` | Home page, authentication hub. |
| `/login`, `/auth/callback`, `/logout` | Microsoft Entra ID authentication flow. |
| `/dev/login` | Development login: user picker on `GET`, login on `POST` (only with `AUTH_MODE=dev`, see [Development login](#development-login)). |
| `/dashboard/student` | Student dashboard. |
| `/dashboard/program-manager` | Programme manager dashboard. |
| `/dashboard/facilitator` | Facilitator dashboard. |
| `/dashboard/campus-manager` | Campus manager dashboard. |
| `/dashboard/teachers/analytics` | Satisfaction score per teacher, filterable by school year / semester / programme. Available to the `campus_manager` and `program_manager` roles, scoped to each one's perimeter. |
| `/dashboard/admin` | Administrator dashboard. |
| `/dashboard/survey-create` | Survey creation and settings. |
| `/api/surveys/{survey_id}` | Questionnaire (answering the survey). |
| `/api/surveys/{survey_id}/status` | Survey status change. |
| `/api/surveys/{survey_id}/students` | Management of the students enrolled in a survey. |
| `/api/surveys/{survey_id}/export` | CSV export of the answers. |
| `/api/surveys/{survey_id}/visualisation` | Answer visualisation. Accepts `?teacher=<name>` to open already filtered on a teacher. |
| `/api/surveys/{survey_id}/generate-summaries` | Starts LLM summary generation. |
| `/api/surveys/{survey_id}/destroy-summaries` | Deletes the generated summaries. |
| `/api/users/{user_id}/role` | Changes a user's role. |
| `/backend/prompts` | LLM prompt list (admin only). |
| `/backend/prompts/new` | Prompt creation form. |
| `/backend/prompts/{id}/edit` | Prompt edit form. |
| `/api/prompts` | Creates a prompt (POST, form). |
| `/api/prompts/{id}` | Updates a prompt (PUT, fetch). Blocked if the prompt is referenced in `summaries`. |
| `/api/prompts/{id}/delete` | Deletes a prompt (POST, form). Blocked if the prompt is referenced in `summaries`. |

---

## Running in production

- **Without Docker**: `launch.sh` creates a `venv/` if needed, then runs `python main.py` (Uvicorn on port 8000, no reload) and `summaries_generator_daemon.py` in two separate `screen` sessions, logging to `app.log` / `app.error` and `summaries.log` / `summaries.error`. It expects the project at `/home/mde-admin/OceENS`.
- **With Docker Compose**: `docker compose up --build -d`. `.env` is passed through `env_file` and never copied into the image (it is listed in `.dockerignore`). Set `RUN_SUMMARIES_DAEMON=1` to run the summaries daemon in the same container.

To use an existing database instead of the demo data, place `db_oceens.db` in the database folder (`LOCAL_DATABASE_DIR`) before the first start: the demo data is only inserted when the database has no user. Programmes (`import/Program_list.csv`), the default LLM provider and the known model prices are synchronised at every start.

### Summaries daemon

`summaries_generator_daemon.py` generates the summaries requested from the interface: it polls the database, calls the LLM provider one request at a time, and writes the result back. Without it, requested summaries stay pending. Start it with `python summaries_generator_daemon.py` (with the virtual environment's interpreter), or set `RUN_SUMMARIES_DAEMON=1` so the application starts it. It loops, writes to the database and contacts an external LLM service: only run it when needed.

---

## Logging

Application logs use Python's standard `logging` module and the `uvicorn` logger. Messages from the application, `auth.py` and `seed.py` thus reuse the format, colours and handlers already configured by the server.

Levels are used according to severity:

| Level | Use |
|-------|-----|
| `DEBUG` | Detailed information useful for development and seeding. |
| `INFO` | Startup, shutdown and normal application operations. |
| `WARNING` | Expected resource missing, or non-blocking situation. |
| `ERROR` / `EXCEPTION` | An operation failed; `logger.exception()` keeps the traceback. |
| `CRITICAL` | Essential configuration missing, preventing startup. |

Example:

```python
import logging

logger = logging.getLogger("uvicorn")

logger.info("Operation complete")

try:
    risky_operation()
except Exception:
    logger.exception("Operation failed")
```

New diagnostics should use the appropriate logger rather than `print()`. The application level is currently set to `DEBUG` in `core/dependencies.py`. Application logs go through the Uvicorn handler, usually written to `stderr`; with separate redirection, use for example `2> error.log` to capture them.

---

## LLM providers (free-text summaries)

Free-text summaries are generated by an LLM. The provider is **configurable from the interface** (`/backend/providers`, admin only), without touching the code. The default provider is **Ollama EPF** (`https://locallm.mde.epf.fr/ollama`), created automatically at startup; its key is read from `LLM_API_KEY`. Each student gets their own key from <https://locallm.mde.epf.fr> by logging in with their EPF account.

### Supported API types

| `api_type` | Covers |
|------------|--------|
| `ollama`    | Ollama servers (local, EPF, third-party) |
| `openai`    | OpenAI **and any OpenAI-compatible endpoint**: vLLM, Groq, Mistral, LM Studio… |
| `anthropic` | Claude API (Anthropic) |

### Security principle: no key in the database

The SQLite database is not encrypted and ends up in backups. **No API key is therefore stored in it.** The `llm_providers` table only holds the *name* of the environment variable (`api_key_env`, e.g. `OPENAI_API_KEY`); the value stays in `.env` and is only resolved at call time. That name is checked against an allow list (`LLM_*` or `*_API_KEY`) to prevent pointing at a system secret (`SECRET_KEY`, `ENTRA_CLIENT_SECRET`…).

### Adding a provider

1. **Add the key to `.env`** with a compliant name (`LLM_*` or `*_API_KEY`):

   ```env
   OPENAI_API_KEY=sk-...
   ```

2. **Restart the summaries daemon** (`.env` variables are only read at startup).

3. **Create the provider** in `/backend/providers` → *+ Nouveau fournisseur*: enter the name, the API type, the base URL, the name of the environment variable (`OPENAI_API_KEY`) and a default model. The key present / missing indicator confirms that the variable is loaded. The *Tester* button checks that the URL and the key respond, then sends a one-token generation to confirm that the account can actually generate (see below).

4. **Link a prompt** to the provider: in `/backend/prompts`, a `<select>` chooses a prompt's provider. A prompt without a provider (`provider_id` NULL) falls back to Ollama EPF.

> [!NOTE]
> A provider referenced by at least one prompt cannot be deleted (so as not to break those prompts' configuration).

### Exhausted credit and other provider errors

Each provider reports failures in a different format: exhausted credit is a `429 insufficient_quota` at OpenAI, but a `400 "Your credit balance is too low"` at Anthropic. `services/llm_client.py` normalises these responses into categories (`quota`, `rate_limit`, `auth`, `model`, `server`) and derives a readable message from them, in French like the rest of the interface, for example:

> ⚠️ Crédit ou quota épuisé chez le fournisseur : la clé est valide mais le
> compte ne peut plus générer. Rechargez le compte ou choisissez un autre
> fournisseur. (fournisseur OpenAI, modèle gpt-4o-mini, HTTP 429)

This message is written to `Summary.metadata_text` instead of the raw JSON, so it is visible directly from the interface when a summary fails. The provider's raw response stays in the daemon logs for diagnosis.

> [!IMPORTANT]
> The *Tester* button does not just list the models: at OpenAI as at Anthropic, `GET /v1/models` still answers perfectly with a zero balance. A one-token generation ping (negligible cost) is therefore sent next: it is the only way to detect exhausted credit **before** starting a summary campaign.

---

## Summary costs

The cost of each summary is **measured, not estimated**. At generation time, the daemon records the token counters returned by the provider (`Summary.input_tokens`, `output_tokens`, `model_used`): it is the only chance to capture them, no API lets you ask for them afterwards. The amount is then obtained by crossing these counters with the price list.

> [!NOTE]
> This section replaces the former `llm-utils/token-counting/` scripts, which counted the tokens of the **repository's source code** and multiplied them by a hard-coded price. That measurement said nothing about the application's real spending. Tracking now covers the calls actually billed.

### Price list: `/backend/llm/prices`

Prices live in the database (`llm_model_prices` table), **in euros**. They are editable from the administration: no release is needed to follow a price revision, nor to cover a locally added provider. A price has two components, added together:

- a **flat fee per generation**, as a range (`flat_cost_min` / `flat_cost_max`; an exact fee has min = max), for models whose cost is not measured in tokens;
- a **price per million tokens**, input and output, for commercial providers that bill by consumption.

Every cost is therefore a `(min, max)` range, shown as a single amount when both ends are equal.

A price can be entered in euros or in dollars, as providers publish them: dollars are converted once, at entry, with the dollar → euro rate stored in the database (0.92 by default, editable on the same page). Changing the rate later does not rewrite existing prices, so costs already computed do not change retroactively.

Pre-filled at startup (`seed_model_prices`, idempotent: a price corrected by hand is never overwritten):

| Model | Flat fee per generation | Input / Output per M tokens |
| --- | --- | --- |
| `gemma4:26b` (Ollama EPF, self-hosted) | €0.02 to €0.05 (GPU, power, hardware amortisation) | – |
| `claude-opus-5` | – | $5.00 / $25.00, converted to euros |
| `claude-sonnet-5` | – | $3.00 / $15.00, converted to euros |
| `claude-haiku-4-5` | – | $1.00 / $5.00, converted to euros |

Prices of other providers (OpenAI, Mistral, Groq…) **must be entered**: they are not guessed. A provider-specific price wins over a generic price with the same model name.

### Where to look

| Where | What |
| --- | --- |
| `/backend/llm/costs` | Overall cost, broken down by survey and by model (admin) |
| 💰 button on a survey row | Cost of that survey's summaries |

### What is not priced

A summary cannot be priced when its counters are missing (generated before this feature, or a provider that does not expose them) or when its model has no recorded price. It is then **counted separately**, never estimated or brought down to zero: an invented amount would be more harmful than a missing one, since it would be displayed with the authority of a real amount. Screens explicitly say when a total is partial.

Not to be confused with a **zero** cost, which is a real amount and not the same information as "unknown". A self-hosted model is not free either: the school's server costs a flat fee per generation.

> [!IMPORTANT]
> Tracking starts when the feature goes live: summaries generated earlier have no counters in the database and cannot be priced retroactively.

---

## Project structure

```
OceENS/
├── main.py                       # FastAPI factory, middlewares and router assembly
├── sondage_loader.py             # Loads a full survey for export
├── survey_loader_from_xlsx.py    # Imports surveys from an Excel file
├── summaries_generator_daemon.py # Asynchronous LLM summary processing (separate process)
├── launch.sh                     # Launch script (production, without Docker)
├── requirements.txt              # Python dependencies
├── Dockerfile                    # Application Docker image
├── docker-compose.yaml           # Docker Compose service (reads .env)
├── .dockerignore                 # Files excluded from the Docker build
├── .env.example                  # Environment variable template, copied to .env
├── .env                          # Environment variables (⚠️ not committed)
├── .gitignore                    # Files and folders ignored by Git
├── CONTEXT.md                    # Domain glossary and documentation language
│
├── core/                         # Low-level access and security
│   ├── auth.py                   #   Microsoft Entra ID authentication (login, logout, callback) and development login
│   ├── database.py               #   SQLite engine and SessionDep dependency
│   ├── security.py               #   Roles, scopes, access control
│   ├── dependencies.py           #   Shared Jinja templates and logger
│   └── seed.py                   #   Initial data and programme synchronisation
│
├── models/                       # SQLModel schema, one file per table
│   ├── __init__.py               #   Re-exports every class (see its docstring)
│   └── User.py, Survey.py, ...
│
├── routers/                      # Routes split by business domain
│   ├── pages.py                  #   Home page and per-role dashboards
│   ├── surveys.py                #   Surveys: CRUD, status, export, visualisation
│   ├── students.py               #   Student enrolment in a survey
│   ├── users.py                  #   User role management
│   ├── summaries.py              #   Triggers LLM summaries
│   ├── prompts.py                #   Prompt administration
│   ├── survey_templates.py       #   Survey template administration
│   ├── sections_questions.py     #   Section and question administration
│   └── llm/                      #   LLM administration
│       ├── _access.py            #     Shared access control for the LLM screens
│       ├── providers.py          #     LLM providers (CRUD + connection test)
│       ├── prices.py             #     Price list per model
│       └── costs.py              #     Overall and per-survey cost
│
├── services/                     # Business logic
│   ├── helpers.py                #   Navigation, statistics, filters, sorting
│   ├── visualisation_data.py     #   Aggregations and visualisation context
│   ├── llm_client.py             #   Multi-provider LLM client (ollama/openai/anthropic)
│   ├── llm_costs.py              #   Summary cost (measured tokens × price list)
│   ├── settings_store.py         #   Application settings stored in the database
│   └── export_csv.py             #   CSV export of answers
│
├── import/                       # Seed data: programme list and demo answers (CSV)
├── database/                     # SQLite database folder (ignored by Git)
│   └── db_oceens.db
│
├── docs/
│   ├── smoke-test.md             # Manual smoke test
│   ├── adr/                      # Architecture decision records
│   └── agents/                   # Instructions for coding agents
│
├── llm-utils/                    # LLM tooling outside the application
│   └── README.md                 # (cost tracking moved into the app, see above)
│
├── templates/                    # HTML templates (Jinja2)
│   ├── index.html                     # Home / login page
│   ├── dashboard/
│   │   ├── admin.html
│   │   ├── student.html
│   │   ├── program_manager.html
│   │   ├── facilitator.html
│   │   ├── campus_manager.html
│   │   ├── teachers-analytics.html       # Teacher satisfaction (campus_manager, program_manager)
│   │   ├── survey.html                   # Answering a survey
│   │   ├── survey_create.html            # Survey creation
│   │   └── visualisation.html            # Answer visualisation
│   ├── backend/                       # Administration pages (admin only)
│   │   ├── prompts.html               # LLM prompt list
│   │   ├── prompt_form.html           # Shared create/edit form
│   │   └── llm/                       # LLM screens (providers, prices, costs)
│   │       ├── providers.html
│   │       ├── provider_form.html
│   │       ├── prices.html            # Editable price list
│   │       └── costs.html             # Overall and per-survey cost
│   └── template_parts/                # Fragments reused across dashboards
│       ├── part_site_header.html
│       ├── part_dashboard_navigation.html
│       ├── part_theme_switcher.html
│       └── ...
│
├── static/
│   ├── css/                      # admin.css, student.css, program_manager.css, survey.css,
│   │                              # survey_create.css, visualisation.css, prompt_form.css,
│   │                              # llm_backend.css (LLM screens), theme.css, site_header.css,
│   │                              # dashboard_navigation.css, responsive.css
│   ├── js/
│   │   └── survey.js
│   └── img/
│
└── .venv/                        # Python virtual environment (not committed)
```

---

## Authentication (OAuth 2.0)

With `AUTH_MODE=entra`, authentication relies on **Microsoft Entra ID** through the MSAL library:

```
1. The user clicks "Se connecter"
   → FastAPI generates a random state (UUID, CSRF protection)
   → Redirect to the Microsoft login page

2. The user authenticates with Microsoft
   → Microsoft redirects to /auth/callback with a code + state

3. The server exchanges the code for an access token
   → Fetches the user's details through Microsoft Graph
   → Looks up the user's role(s) and scope in the database
   → Creates the session {name, email, roles}
   → Redirects to the matching dashboard

4. On logout (/logout)
   → Deletes the session and cookies
   → Logs out on the Microsoft side
   → Back to the home page
```

Authentication alone authorises no business action: every route then checks the role and scope (programme or campus) through `require_roles()` and the related helpers.

---

## Development login

To work on a fork without an Azure application, the **development login** (`AUTH_MODE=dev`) lets you log in as any user, with no proof of identity. It must **never** be used in production. Its variables (`AUTH_MODE`, `DEV_LOGIN_KEY`, `SECRET_KEY`, `ALLOWED_DOMAINS`) are described in the [configuration reference](#configuration-reference).

In `dev` mode, the session cookie is no longer restricted to HTTPS (`http://localhost` works), `/login` redirects to `/dev/login`, `/auth/callback` does not exist and `/logout` clears the session then redirects to `/`. A warning is logged at startup. A red, non-dismissible banner is shown at the top of every page that includes the shared header: it shows the logged-in address, offers *Changer d'utilisateur* (`/dev/login`) and says *accès ouvert à tous* when `DEV_LOGIN_KEY` is not set.

`POST /dev/login` expects a form with `email`, `name` (optional) and `key` (if `DEV_LOGIN_KEY` is set). The user is fetched or created as on return from Entra: an unknown address becomes a new student. Without `name`, the display name is built from the address (`bob.leponge@epfedu.fr` → "Bob Leponge"). A new login replaces the session: that is how you switch users.

In a browser, `GET /dev/login` lists the database's users, grouped by role name without scope (a user with no role appears under `student`, a user with several roles under each of them). A click logs in as the chosen user; a free field accepts another address, with an optional name. If `DEV_LOGIN_KEY` is set, a single key field is shown and used for every login on the page; the key is never stored in the session. Come back to this page to switch users.

```bash
AUTH_MODE=dev DEV_LOGIN_KEY=my-key uvicorn main:app

# Log in as the seed admin; -c stores the session cookie
curl -i -c cookies.txt \
  -d email=antoine.gademer@epf.fr -d key=my-key \
  http://localhost:8000/dev/login

# Reuse the cookie (-b) for the following requests
curl -b cookies.txt -c cookies.txt -L http://localhost:8000/
```

---

## Notable features

### Teacher analytics

The `/dashboard/teachers/analytics` route (`campus_manager`, `program_manager`) aggregates the satisfaction score per `(teacher, survey)` from the `QCU_Satisfaction` answers that carry an `Answer.teacher` (ME sections). The teacher list is sorted with `teacher_sort_key()`, case- and accent-insensitive, and can be filtered by school year, semester, programme and teacher.

### Teacher filter in the visualisation

A client-side selector filters the visualisation without reloading: only the chosen teacher's modules remain visible, the Campus and Programme sections being hidden. The page reads `?teacher=<name>` on load to pre-filter itself; links from the analytics pass this parameter, so clicking a teacher's score opens their view directly.

### Surveys imported from Excel

Surveys loaded by `survey_loader_from_xlsx.py` have no `QCU_Attendance` question: `services/visualisation_data.py` then uses `satisfaction_responses_count` as the fallback denominator for the teacher score. Teacher names are normalised with `.title()` on import as on aggregation, to merge case variants (`"GADEMER Antoine"` and `"Gademer Antoine"` = a single entry). Questions are sorted by `question_id` in the template, which puts charts before free-text answers whatever the insertion order.

### Campus manager scope

The `campus_manager` dashboard only shows closed surveys with at least one respondent. The link to the questionnaire and the QR code are hidden there (`can_view_survey_link=False`): this role reads results without distributing surveys. The `{% if can_view_survey_link | default(true) %}` guard leaves the other dashboards unchanged.

### Orphan student cleanup

When a survey is deleted, students no longer attached to **any other** survey are deleted too, to avoid piling up unused accounts (`services/helpers.py`, `_delete_orphan_students`). A safeguard protects users with a privileged role (`admin`, `program_manager`, `facilitator`, `campus_manager`): a teacher or manager who answered a survey is never deleted.

### Adding a user by e-mail

The administrator dashboard's *Utilisateurs* tab has a *+ Ajouter un utilisateur* button: an e-mail address is enough to create the account, with the `student` role by default (`POST /api/users`, admin only). The address is validated (format + allowed domain) and duplicates are refused.

---

## Deployment checklist

- [ ] `AUTH_MODE` unset or `entra`
- [ ] `ENTRA_CLIENT_ID`, `ENTRA_CLIENT_SECRET`, `ENTRA_TENANT_ID`, `REDIRECT_URI` and `ALLOWED_DOMAINS` set with the real Entra application's values
- [ ] A dedicated `SECRET_KEY` (see the [configuration reference](#configuration-reference))
- [ ] Valid SSL certificate (Let's Encrypt or equivalent): outside `dev` mode, session cookies are HTTPS-only
- [ ] Database present in the database folder (`LOCAL_DATABASE_DIR`), or Docker volume mounted
- [ ] Secrets, including `LLM_API_KEY`, kept out of Git and out of the image (`.env` through `env_file`)
- [ ] Summaries daemon running if LLM summaries are used (`launch.sh` or `RUN_SUMMARIES_DAEMON=1`)

---

## Before contributing

The repository has no automated test suite and no CI yet. Before proposing a change, run the [manual smoke test](docs/smoke-test.md): its static checks for every change, and its startup steps for any change to startup, configuration, dependencies or the container.

---

## Resources

- [FastAPI](https://fastapi.tiangolo.com/)
- [FastAPI and Uvicorn logging guide](https://apitally.io/blog/fastapi-logging-guide)
- [MSAL Python](https://github.com/AzureAD/microsoft-authentication-library-for-python)
- [Microsoft Graph](https://learn.microsoft.com/en-us/graph/)
- [Jinja2](https://jinja.palletsprojects.com/)
- [SQLAlchemy](https://www.sqlalchemy.org/)
- [SQLModel](https://sqlmodel.tiangolo.com/)
- [Pandas](https://pandas.pydata.org/)

---

**OcéEns team**, EPF
