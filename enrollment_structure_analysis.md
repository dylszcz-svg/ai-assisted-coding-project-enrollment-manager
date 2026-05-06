# Enrollment Structure Analysis

## Overview

`enrollment_starter.py` is an intentionally procedural backend for a student enrollment app.
It works, but functions mix database responsibilities with service-layer logic, module-level
constants are unorganized, and some functions have more than one reason to change.
This document records the structural issues identified before refactoring begins.

---

## Structural Issues

| Function / Responsibility | Issue | Why It Hurts |
|---|---|---|
| `DB_PATH`, `SNAPSHOT_PATH`, `STATUS_ENROLLED`, `CURRENT_STUDENT` | Module-level globals scattered at the top with no organization | Any file that imports this module picks up all constants, including ones it doesn't need. Hard to swap out later (e.g. different DB path per environment). |
| `AVAILABLE_COURSE_KEYS` | Course catalog hardcoded as a Python list, not read from the DB | The DB has a `courses` table with the same data — two sources of truth that can drift out of sync. |
| `seed_sample_data` | Reads `AVAILABLE_COURSE_KEYS` global directly instead of receiving it as a parameter | The DB layer is reaching into module-level config to do its job. Makes it impossible to seed different data without changing the global. |
| `enroll_with_key` | Validates the key, calls `get_course_by_key` (DB), then writes the enrollment (DB) | Business logic is tangled with raw SQL writes. One function owns too many decisions — validation and persistence should be separated. |
| `get_student_summary` | Calls `get_student_enrollment_history` (a DB function) and counts results in Python | This is the only function that belongs in the service layer — it interprets data rather than fetching it — but it lives alongside DB functions with no distinction. |
| `export_database_snapshot` | Queries two DB tables, assembles a dict, then writes JSON to disk | Three responsibilities in one function: querying, assembling a snapshot structure, and file I/O. Each belongs in a different layer. |
| `connect` | Returns a new connection on every call; no connection is stored or reused | Every function opens and closes its own connection. Once this becomes a class, connection creation should be owned consistently by `EnrollmentStore`. |
| `main` | Calls DB functions, service functions, and file export all in sequence | Acts as a makeshift orchestrator. In the refactored version this responsibility moves to `app.py` or a dedicated runner. |
| `rows_to_dicts` | Utility that lives at module level with no clear home | Belongs inside `EnrollmentStore` since it only exists to clean up SQLite row objects. |
| `SQLite SELECT, INSERT, UPDATE` | Raw SQL is spread across many standalone functions with no encapsulation | Any schema change (e.g. renaming a column) requires hunting through every function. Encapsulating in a class makes changes easier to find and test. |

---

## Cross-Layer Violations (Highest Priority)

### `enroll_with_key`
The clearest cross-layer problem. The validation question — "does this key match a real course?" — is
service logic. The write — "insert or update the enrollment row" — is DB logic. Right now one function
does both. If enrollment rules change (e.g. max enrollments per student), you'd be editing a function
that also owns SQL.

**Future design:** Split into a service method that validates and delegates, calling separate DB methods
for the lookup and the write.

### `export_database_snapshot`
Queries two DB tables, builds a Python dict, and writes a JSON file — three separate jobs glued
together in a script context.

**Future design:** DB queries stay in `EnrollmentStore`. The dict assembly and file write move to a
utility or service method.

### `get_student_summary`
Calls a DB function (`get_student_enrollment_history`) and counts results in Python. The counting
and interpretation logic is service-layer work masquerading as a DB function.

**Future design:** Move to `EnrollmentManager` (service class). It calls the store for raw data,
then applies its own logic.

---

## Layer Design Map

| Future Layer | Functions That Belong There |
|---|---|
| Constants / config | `DB_PATH`, `SNAPSHOT_PATH`, `STATUS_ENROLLED`, `STATUS_UNENROLLED`, `CURRENT_STUDENT`, `AVAILABLE_COURSE_KEYS` |
| Database class (`EnrollmentStore`) | `connect`, `create_tables`, `seed_sample_data`, `get_available_course_keys`, `get_course_by_key`, `get_student_enrollments`, `get_student_enrollment_history`, `get_student_course_record`, `soft_unenroll_student`, `get_all_enrollment_records`, `rows_to_dicts`, all raw SQL |
| Service class (`EnrollmentManager`) | `get_student_summary`, enrollment validation portion of `enroll_with_key` |
| Needs splitting | `enroll_with_key`, `export_database_snapshot`, `main` |
