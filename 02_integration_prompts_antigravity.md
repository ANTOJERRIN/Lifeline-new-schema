# Pharmacy tier — integration prompts for Antigravity

Run these in order. Each one is Role / Task / Format. Paste Step 0 before
every single prompt below — it's what stops Antigravity from touching
anything outside the pharmacy tier.

Verified against the real repo: `ANTOJERRIN/lifeline-backend`, `backend/app/*`.

---

## Step 0 — paste this before every prompt, every time

```
GUARDRAILS:

1. Supabase in this project is Postgres tables + one storage bucket
   (medical-reports) only. There is no Supabase Auth usage anywhere in this
   codebase — do not add any. Auth is 100% custom JWT + bcrypt in
   app/auth/security.py. Nothing about auth changes except adding one new
   role check function, in Step 4 below.

2. You may ONLY create new files under a new app/pharmacy/ directory, plus
   one new migration file, migrations/005_pharmacy.sql.

3. The ONLY existing files you are allowed to edit, and only in the exact
   steps named below, are:
   - app/main.py           (Step 6 — add 1 import line + 1 include_router line)
   - app/auth/security.py  (Step 4 — add 1 new function only)

4. Never touch: app/database/supabase.py, app/config/settings.py, any .env
   file, or any existing route/service/schema file not listed above.

5. The migration is additive only: new tables, new nullable columns. Never
   ALTER a column type, never DROP anything, never rename an existing table
   or column.

6. Before writing any code in any step, list every file you're about to
   create or touch. If that list has anything not covered by points 2-3
   above, stop and tell me instead of proceeding.
```

---

## Step 1 — Read only, no code

**Role:** Senior FastAPI engineer, new to this codebase.

**Task:**
1. Open and read `app/consent/routes.py`, `app/consent/service.py`,
   `app/consent/schemas.py` end to end.
2. Open and read `app/auth/security.py` end to end.
3. Open and read `app/services/audit_service.py` end to end.
4. Write down, in plain sentences:
   - How `require_doctor` / `require_staff` / `require_patient` work.
   - What `check_doctor_consent()` in `app/services/medical_record_service.py`
     actually checks (read that function too).
   - The exact call signature of `write_access_log()`.

**Format:** Prose summary only. No code. End with a numbered list of every
file you now expect to create in Steps 2-6, for me to check against the
guardrails before you continue.

---

## Step 2 — Write the migration

**Role:** Backend engineer writing a Supabase (Postgres) migration.

**Task:** Create `migrations/005_pharmacy.sql` with exactly these two new
tables (do not add anything else without asking first):

```sql
CREATE TABLE prescription_status (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    medical_record_id UUID NOT NULL REFERENCES medical_records(id),
    status TEXT NOT NULL DEFAULT 'prescribed',  -- 'prescribed' | 'sent_to_pharmacy' | 'dispensed'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dispense_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prescription_status_id UUID NOT NULL REFERENCES prescription_status(id),
    pharmacist_id BIGINT NOT NULL REFERENCES users(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    items JSONB NOT NULL DEFAULT '[]',
    dispensed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Also write a matching `migrations/apply_005.py`, copying the exact pattern
used in `migrations/apply_004.py` (same connection setup, same style).

**Format:** Two files only — the `.sql` and the `apply_005.py`. No
application code yet. Run `apply_005.py` against the dev database and paste
the output before moving to Step 3.

---

## Step 3 — Build the pharmacy module

**Role:** Same engineer, following the exact three-file pattern used in
`app/consent/`.

**Task:** Create these four files:

- `app/pharmacy/__init__.py` — empty, matching every other module.
- `app/pharmacy/schemas.py` — Pydantic models for a prescription list item
  and a dispense request, modeled on `app/consent/schemas.py`.
- `app/pharmacy/service.py` — all Supabase calls live here, none in routes.py.
  Needs:
  - `get_prescriptions_for_patient(patient_id)` — reads `prescription_status`
    joined with the medicines on the linked `medical_records` row.
  - `check_pharmacy_consent(pharmacist_id, patient_id)` — same shape as
    `check_doctor_consent()`, checking `consent_requests` for an approved,
    unexpired row.
  - `dispense(prescription_status_id, pharmacist_id, patient_id, items)` —
    inserts a `dispense_events` row, updates `prescription_status.status` to
    `'dispensed'`, calls `write_access_log(actor_role="pharmacist",
    action="dispensed", actor_user_id=pharmacist_id, patient_id=patient_id,
    metadata={...})`.
- `app/pharmacy/routes.py` — two endpoints only:
  - `GET /pharmacy/prescriptions?patient_id=...` — pharmacist-only, calls
    `check_pharmacy_consent` first, 403 if it fails.
  - `POST /pharmacy/dispense` — pharmacist-only, same consent check.

**Format:** Show the full contents of all four files. Do not touch
`app/main.py` or `app/auth/security.py` in this step.

---

## Step 4 — Add the one new auth function

**Role:** Same engineer, making the single permitted edit to
`app/auth/security.py`.

**Task:** Add exactly one function, `require_pharmacist`, copied in
structure from `require_doctor` (same shape: read `current_user.get("role")`,
403 with a clear message if it isn't `"pharmacist"`). Do not change any other
function in this file.

**Format:** Show a diff. It should be a pure addition — no existing lines
should show as changed or removed.

---

## Step 5 — Wire it into main.py

**Role:** Same engineer, making the single permitted edit to `app/main.py`.

**Task:** Add one import line
(`from app.pharmacy.routes import router as pharmacy_router`) and one
`app.include_router(pharmacy_router)` line, placed after the existing
`app.include_router(appointments_router)` line, matching the existing style
exactly.

**Format:** Show a diff of `app/main.py` only. It should be exactly two new
lines.

---

## Step 6 — Tests

**Role:** Same engineer, writing tests consistent with the existing `tests/`
folder.

**Task:** Look at how `tests/` currently tests `app/consent/` (find the
matching test file), and write the same shape of tests for
`app/pharmacy/`:
1. Non-pharmacist role hitting `/pharmacy/prescriptions` gets 403.
2. Pharmacist with no approved consent gets 403.
3. Pharmacist with approved consent gets the prescription list.
4. `POST /pharmacy/dispense` succeeds, updates `prescription_status`, and
   writes exactly one `access_logs` row.

**Format:** One new test file. Run it
(`pytest tests/test_pharmacy.py -v`) and paste the actual output. Do not
modify any existing test file.

---

## Step 7 — Final check before calling it done

**Role:** Reviewing your own change as if it were someone else's pull request.

**Task, answer each explicitly:**
1. List every file created and every file touched, one line each on why.
2. Confirm nothing outside `app/pharmacy/`, `migrations/005_pharmacy.sql`,
   `app/main.py`, and `app/auth/security.py` was touched.
3. Re-read the migration — confirm there is no `ALTER ... TYPE`, `DROP`, or
   rename statement in it.
4. Confirm both new endpoints check role AND consent before touching any
   patient data.
5. Confirm `write_access_log` is called on the dispense path.

**Format:** Answer all five in order. If any answer is "no" or "not sure,"
stop and flag it instead of telling me it's done.
