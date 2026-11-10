# osce-scoring-system
# Radiotherapy OSCE Scoring and Quality Assessment System

A semi-automated scoring and evidence-generation system for radiotherapy residency
training (OSCE): candidates submit a treatment plan package, the system computes
contour geometry, dosimetric and similarity metrics against preset rules, converts
them into Station D / Station E scores, and emits a traceable scoring report.

- **Station D** (cervical cancer target volume and pelvic OAR contouring, irradiation extent and prescription) - maximum score **30 points**, 10 items
- **Station E** (target coverage, pelvic OAR dose, dose distribution and radiotherapy QA) - maximum score **10 points**, 4 items
- **Total** - maximum score **40 points**
- Backend FastAPI + SQLite, frontend plain HTML / JS, **no Node.js required, fully offline-capable**

> **A semi-automated scoring system for radiotherapy OSCE.** Trainees submit a
> treatment plan package (CT / RTSTRUCT / RTPLAN / RTDOSE); the system computes
> contour geometry, dosimetric and similarity metrics, converts them into
> station-level scores (D 30 + E 10 = 40), and produces an auditable report.
> Backend: FastAPI + SQLAlchemy + SQLite. Frontend: plain HTML/JS. Offline-capable.

---

## ⚠️ Disclaimer

1. **Rule parameters must be reviewed on your own**: every rule file under `config/` carries `approval_status` and `note` fields.
   The cervical cancer main templates (`reference_plan_rules.yaml`, `scoring_rules.yaml`) are marked
   `approved` according to the original authors' expert panel; while the `note` fields of
   `contour_scoring_rules.yaml` and `contour_integration_rules.yaml` still read
   "example rules, pending expert confirmation", coexisting with the `approval_status` in the file header - the
   authoritative wording for such rules is whatever your own center's expert review concludes.
   Before using this in another institution, review every item, bump `rule_version`, then freeze the version.
   **These thresholds are scoring criteria, not clinical treatment recommendations.**
2. **No patient data**: this repository contains only program code, scoring rules and documentation. It
   **contains no DICOM data, no real case information, no database files and no exported reports**.
   The sample data is synthesized on the fly by the scripts under `examples/`.
3. **Assess compliance yourself**: for real examination use, complete ethics review, data security
   review and software approval according to your institution and local regulations.

---

## 1. Quick Start

### One-step deployment on Windows

```bat
:: 1) Create the virtual environment and install dependencies (double-clicking also works)
setup_env.bat

:: 2) Start (the login page opens automatically once ready)
start.bat
```

### Generic method (any platform)

```bash
python -m venv .venv
.venv/bin/python -m pip install -r requirements.txt      # Windows: .venv\Scripts\python.exe

.venv/bin/python -m scripts.init_db                      # create tables + initial administrator + 3-month pre-authorization
.venv/bin/python -m uvicorn app.main:app --host 0.0.0.0 --port 8010
```

Open <http://127.0.0.1:8010/login>.

| Item                   | Value                                                        |
| ---------------------- | ------------------------------------------------------------ |
| Default port           | 8010 (`set OSCE_PORT=8020` to change it)                     |
| Initial account        | `admin` / `123456` - **first login only; a password change is enforced afterwards** |
| Database               | `data/osce.db` (single SQLite file, created automatically on first start) |
| Backup directory       | `backup/` (triggered by `backup_data.bat` or from the system settings page) |
| Environment self-check | `check_environment.bat` / `scripts/netinfo.py`               |
| LAN access             | inbound TCP 8010 must be allowed: `run-allow-firewall.bat` (administrator) |

> The first start creates a 3-calendar-month pre-authorization; see `docs/license_management.md`
> for the licensing module. If you only use the system locally, `set OSCE_HOST=127.0.0.1`.

---

## 2. Features

| Module                              | Capability                                                   |
| ----------------------------------- | ------------------------------------------------------------ |
| Case management                     | Create/edit/filter/archive, unique ID validation, state machine, prescription information |
| Station D scoring                   | Item-by-item scoring for D1-D10 plus anchors and deduction rationale; automatic arithmetic check of the D9 prescription |
| Station E scoring                   | Item-by-item scoring for E1-E4 plus issues and comments      |
| Automatic contour comparison        | Single-file / folder / ZIP batch import; ROI matching (DSC, HD95, ASSD, SurfaceDice, RVE); automatic scoring converted into Station D/E scores |
| Reference plan library              | 1-5 expert plans per case; approval status and enable/disable control; only plans that are "qualified and enabled" take part in scoring |
| Multi-reference intelligent scoring | Each student plan is compared against every reference plan: geometry + dosimetric + similarity components; three aggregations (best match / Top-K / weighted) shown side by side; 12 clinical hard constraints applied independently |
| Plan splitting and summation        | Multiple RTPLANs in the same Study are split into independent candidates automatically; multiple plans on the same CT grid can be summed voxel by voxel (shape/spacing mismatches are rejected outright, without interpolation) |
| Report export                       | HTML / Excel (5 worksheets) / PDF, including rule version and aggregation strategy |
| Users and licensing                 | scrypt password hashing, server-side sessions, lockout after failures, enforced password change, role permissions, Ed25519 offline license codes |

For technical details see [`docs/`](docs/README.md) (architecture, algorithms, rules, deployment, troubleshooting).

---

## 3. Directory Structure

```
.
├── app/                    Backend (FastAPI)
│   ├── main.py             Application entry point, route registration, security middleware
│   ├── api/                HTTP interface layer (126 endpoints)
│   ├── services/           Business services (cases, scoring, reports, users, rules, linkage)
│   ├── scoring/            Scoring engine and configuration loading
│   ├── reference/          Multi-reference plan evaluation (loading, metrics, constraints, aggregation, summation)
│   ├── contour/            Contour geometry metrics and scoring
│   ├── dicom/              DICOM parsing, spatial mapping, dose grid
│   ├── security/           Sessions, passwords, access guards
│   ├── licensing/          Pre-authorization and license code verification
│   ├── models/ schemas/    ORM models and Pydantic models
│   └── database/           Engine and sessions
├── frontend/               Plain frontend (login.html, index.html, assets/)
├── config/                 Scoring rules, metric rules, ROI aliases, scenario declarations, license public key
├── scripts/                Initialization, batch scoring, environment self-check, operational scripts
├── tests/                  Automated tests (including a synthetic DICOM generator)
├── examples/               Generation scripts for synthetic teaching data
├── docs/                   Design and usage documentation
├── data/                   Runtime database directory (gitignored)
├── requirements.txt        Runtime dependencies
└── setup_env.bat / start.bat / stop.bat / backup_data.bat
```

`data/`, `backup/`, `uploads/` and `logs/` are all **runtime artifacts**, created automatically on first start
and kept out of version control.

---

## 4. Configuration and Environment Variables

All scoring rules live under `config/`; changing rules needs no code changes:

| File                                           | Purpose                                                      |
| ---------------------------------------------- | ------------------------------------------------------------ |
| `scoring_rules.yaml`                           | Station D/E items, maximum scores, anchors, critical safety issues, built-in demo cases |
| `contour_scoring_rules.yaml`                   | Definitions, grouping, thresholds and bands of contour geometry metrics (single source of truth) |
| `contour_integration_rules.yaml`               | Mapping and weights from contour metrics to Station D/E scoring items |
| `reference_plan_rules.yaml`                    | Multi-reference evaluation: metric set, clinical constraints, component weights, aggregation strategy |
| `roi_aliases.yaml` / `roi_aliases_cervix.yaml` | ROI names in the TPS -> canonical names (no guessing, read-only, traceable) |
| `scenarios/cervix.yaml`                        | Disease-site scenario declaration (default scenario)         |
| `excluded_cases.yaml`                          | Case-level exclusion registry (**empty template** in this repository) |
| `osce_license_public.pem`                      | Public key for license code verification (the private key is not in the repository) |

Environment variables:

| Variable                  | Default        | Description                                                  |
| ------------------------- | -------------- | ------------------------------------------------------------ |
| `OSCE_PORT`               | `8010`         | Listening port                                               |
| `OSCE_HOST`               | `0.0.0.0`      | Listening address; set `127.0.0.1` for local-only use        |
| `OSCE_DB_PATH`            | `data/osce.db` | Database file path (used to isolate test and demo databases) |
| `OSCE_REFERENCE_SCENARIO` | `cervix`       | Disease-site scenario                                        |
| `OSCE_PYTHON`             | empty          | Interpreter path used by the start scripts (set it when Python is not in `.venv` or on PATH) |
| `OSCE_AUTH_DISABLED`      | empty          | **For automated tests only**; set to `1` to bypass login validation |

---

## 5. Testing

```bash
python -m pytest tests -q
```

Current baseline: **315 passed / 26 skipped / 0 failed**.
The 26 skipped cases are the ones that need `demo_data/` - that directory holds de-identified copies of real
plan packages and is **not distributed with the repository**; when it is missing the cases are skipped
automatically (they never pretend to pass).

All tests run on DICOM synthesized on the fly by `tests/synthetic_plans.py` (CT + RTSTRUCT + RTDOSE + RTPLAN);
geometry and dose are determined explicitly by parameters, and no real patient information is involved.

Generate a synthetic plan package that can be inspected manually:

```bash
python examples/generate_synthetic_plans.py --out ./example_plans
```

---

## 6. Data and Privacy

- The repository contains **no** DICOM, databases, score sheets or screenshots; `config/excluded_cases.yaml`
  is an empty template and contains no real case IDs.
- At runtime the system reads the plan packages users import. DICOM headers may contain names, IDs,
  institutions and other information, so **de-identify them before import** (this system does not anonymize;
  it only parses and computes).
- `uploads/` holds temporary copies of batch imports; clean them with `python scripts/clean_uploads.py --days 0`.
- Databases, backups and logs all stay on the local disk; the system sends no data outside.

---

## 7. Known Limitations

1. Single-machine SQLite deployment; concurrent users should be <= 3.
2. RAR / 7z are not supported; batch import accepts ZIP only.
3. Station E items E1-E4 do not take part in automatic contour geometry import by default (geometry metrics cannot replace dosimetry and machine QA).
4. License codes are bound to the device; a new machine requires a new code; there is no online revocation.
5. No code signing and no anti-reverse-engineering hardening.
6. Plan summation only adds voxel by voxel on the same grid and rejects shape/spacing mismatches; it never resamples.

---

## 8. Citation

If this system helps your research or teaching, please cite the accompanying paper. The full bibliographic
record of the paper (authors, journal, DOI) will be added to `CITATION.cff` once it is formally published;
until then you can cite this repository:

```
<author>. Radiotherapy OSCE Scoring and Quality Assessment System (Version 1.1)
[Computer software]. 2026. https://github.com/<owner>/<repo>
```

`CITATION.cff` already follows the CFF 1.2 structure; after publication you only need to fill in the journal
fields under `authors` / `preferred-citation` so that GitHub's "Cite this repository" recognizes it correctly.

---

## 9. License

[MIT](LICENSE). **Disclaimer**: this software is provided "as is", without any express or implied warranty;
users are responsible for their own judgment when it is used for clinical or educational decisions.
