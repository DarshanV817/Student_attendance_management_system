# Student Attendance Management System (fixed version)

Flask + MySQL. Admin login, students, subjects, per-subject enrollment, attendance
marking (Present / Absent / Late / Excused), records, reports and CSV export.

## 1. Install
```bash
python -m venv .venv
.venv\Scripts\activate          # Windows      (macOS/Linux: source .venv/bin/activate)
pip install -r requirements.txt
```

## 2. Configure
```bash
copy .env.example .env          # macOS/Linux: cp .env.example .env
```
Open `.env` and set at least:
* `SECRET_KEY` – generate with `python -c "import secrets; print(secrets.token_hex(32))"`
  (the app refuses to start without a strong key)
* `DB_USER` / `DB_PASSWORD` – a MySQL account that can create the database.
  Using `root` with an empty password often fails on MySQL 8; create a user instead:
  ```sql
  CREATE USER 'attendance_user'@'localhost' IDENTIFIED BY 'a-strong-password';
  GRANT ALL PRIVILEGES ON attendance_db.* TO 'attendance_user'@'localhost';
  GRANT CREATE ON *.* TO 'attendance_user'@'localhost';   -- only needed for init_db.py
  ```

## 3. Create the database and an admin
```bash
python init_db.py
python create_admin.py
```
**Upgrading an existing (v1) database?** Back it up, then run `migrate_from_v1.sql`
once instead of `init_db.py`. Existing students are enrolled in all existing subjects,
exactly as v1 behaved, so nothing disappears.

## 4. Run
```bash
python app.py                                   # local use, http://127.0.0.1:5000
waitress-serve --listen=127.0.0.1:8000 app:app  # steadier option for real use
```
Check the database connection any time at `/health`.

## 5. Tests (no MySQL needed)
```bash
python -m unittest discover -s tests -v
```

## What was fixed
**Data not saving / saving in the wrong place**
* The Save form could keep the old subject/date after you changed the filter above it,
  so attendance went to the wrong day while the page looked normal. Saving is now
  blocked until you click *Load students*, and the server only trusts the posted form.
* Dates are validated; future dates are rejected; "today" always uses one time zone
  (`APP_TZ`) instead of mixing Python's date with MySQL's `CURDATE()`.
* Attendance saves run in one transaction (all or nothing) with rollback on error.
* Database errors show a clear 503 page and are logged, instead of a crash or silent loss.
* Messages no longer claim success when nothing changed. Error messages stay on screen.
* Every student used to be pre-set to Present, so one careless click saved false
  records. Students now start unmarked; unmarked ones are skipped (with a warning).
* Connection pooling with autocommit, so pages are fast and never read stale data.

**Security**
* `SECRET_KEY` is required (the old default let anyone forge an admin login).
* CSRF protection on every POST; logout is POST-only; debug mode is off by default.
* Login lockout after 5 failures / 5 min, 8-hour sessions, hardened cookie flags,
  `no-store` cache header, security headers, constant-time password check.
* CSV export guards against spreadsheet formula injection and opens correctly in Excel.

**Data model and features**
* Enrollment table: rosters and percentages are per subject, not "every student in everything".
* Late and Excused statuses (Late counts as attended; Excused is left out of the %).
* Archive/restore instead of destructive delete; permanent delete only for archived rows.
* `marked_by`, `created_at`, `updated_at` on attendance; indexes for the common queries.
* Input length limits and email check; pagination on Records; status filter.

## Known limits
* Login lockout is kept in memory, so it resets on restart and is per-process.
* Put the app behind HTTPS and set `COOKIE_SECURE=1` before using it on a network.
* Bootstrap loads from a CDN, so the pages need internet access.
