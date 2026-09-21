# Backend Security Review — Smart Attendance System
## Introduction

This report covers the backend code for the Smart Attendance system,
focusing on how the API handles authentication, authorization, database
access, and dependencies. The goal was to check whether the security
rules described in the project brief (for example, that a lecturer
should only see their own sections, and that every protected endpoint
must check the user's role on the server) are actually followed in the
code, not just described in the documentation.

Each finding below includes the exact part of the code it came from and
how it was confirmed, so the results can be checked again by anyone else
on the team.

---

## Finding 1: Any Logged-In User Can View Any Section's Student List
**Severity: Critical**
**File:** `server.js`

```javascript
app.get(
  "/api/v1/sections/:sectionId/students",
  authenticateToken,
  async (req, res) => {
    ...
    const result = await pool.query(
      `SELECT u.id, u.student_code, u.name
       FROM enrollment e
       JOIN users u ON u.id = e.student_id
       WHERE e.section_id = $1`,
      [sectionId]
    );
```

This endpoint only checks that the user is logged in
(`authenticateToken`). It does not check the user's role, and it does
not check whether the requesting lecturer/TA is actually assigned to
that section. This means any authenticated user — student, lecturer, TA,
anyone — can see the full name list of students in any section, just by
changing the section number in the URL.

This is a real problem because the same file handles a very similar
situation correctly somewhere else, in the attendance session code:

```javascript
if (Number(timetable.staff_id) !== Number(staffId)) {
  return res.status(403).json({ ... "You are not assigned to this section" });
}
```

So the team clearly knows how to write this kind of check — it just
wasn't added to the student list endpoint.

**How this was confirmed:** Read the full route handler line by line.
Confirmed only one middleware (`authenticateToken`) is applied, and
confirmed there is no comparison against `req.user.role` or
`req.user.userId` anywhere before the database query runs.

**Why it matters for this project:** The brief (ATT-FR-01 and ATT-FR-03)
says a lecturer should only be able to see data for their own sections,
and that this must be enforced by the backend, not just hidden in the
UI. This endpoint breaks that rule directly, and it exposes real student
names and IDs.

**Suggested fix:** Add a role check and an ownership check before
running the query — only allow admins, or a lecturer/TA who is actually
assigned to that section.

---

## Finding 2: A Required Package Is Pinned to a Version That Doesn't Exist
**Severity: High**
**File:** `package.json`

```json
"dotenv": "^18.0.1"
```

The `dotenv` package is used to load the database password and JWT
secret from the `.env` file, so it's an important dependency. The
problem is that version `18.x` of `dotenv` does not exist — the real,
latest version published on npm is `17.4.2`.

**How this was confirmed:** Checked the current published version
history for `dotenv` on the npm registry. The newest real release is
`17.4.2`; no `18.x` version has ever been published.

**Why it matters:** A version number that doesn't exist yet is risky for
two reasons. First, installing the project later could behave
unpredictably once a real version `18.x` is eventually released by the
maintainer, since nobody on the team has reviewed what that future
version will actually contain. Second, this exact pattern — depending on
a version that doesn't exist — is how "dependency confusion" attacks
work: an attacker publishes a fake malicious package under a
plausible-looking future version number, hoping it gets installed by
mistake.

**Suggested fix:** Change the version in `package.json` to a real one
(`^17.4.2`), then reinstall and confirm the correct version is used.

---

## Finding 3: JWT Verification Doesn't Restrict the Signing Algorithm
**Severity: Medium**
**File:** `middleware/authMiddleware.js`

```javascript
const decoded = jwt.verify(
  token,
  process.env.JWT_SECRET
);
```

`jwt.verify()` is called without an `algorithms` option. This means the
library will accept whichever algorithm is stated inside the token
itself, instead of the backend enforcing one specific algorithm.

**How this was confirmed:** Read the function call directly — the
`jwt.verify()` method accepts an options object as its third argument,
and none is passed here.

**Why it matters:** Not specifying the algorithm is a known weak
practice in JWT-based systems. It's a smaller risk here since only one
shared secret is used (not a public/private key pair), but it's still
considered bad practice and is an easy fix.

**Suggested fix:**
```javascript
const decoded = jwt.verify(token, process.env.JWT_SECRET, { algorithms: ["HS256"] });
```

---

## Finding 4: User Role Comes From the Token, Not From the Database
**Severity: Medium**
**File:** `middleware/roleMiddleware.js`

```javascript
if (!allowedRoles.includes(req.user.role)) {
```

The role used for every permission check comes from inside the JWT
itself — it was set once at login time and never checked again against
the current value in the database.

**How this was confirmed:** Read the full `authorizeRole` function.
There is no database query anywhere inside it; the check relies entirely
on data stored inside the token when it was issued.

**Why it matters:** If an admin changes a user's role (for example,
removes lecturer access from an account), any token issued before that
change will still work with the old role until it naturally expires.
Since tokens are set to expire after 1 hour (seen in `server.js`,
`expiresIn: "1h"`), the actual exposure window is limited, but it's still
worth being aware of as a deliberate design tradeoff rather than an
oversight.

**Suggested fix:** No urgent change needed given the short token
lifetime, but this should be documented as a known limitation, or
addressed later with a token-revocation/refresh mechanism if longer
sessions are ever introduced.

---

## Finding 5: No Logging of Failed Login or Permission Attempts
**Severity: Low**
**Files:** `middleware/authMiddleware.js`, `middleware/roleMiddleware.js`

Every failed authentication (`401`) or forbidden (`403`) response is
sent back to the user, but nothing is recorded on the server side about
who attempted it or when.

**How this was confirmed:** Searched both middleware files for `log` or
`console` — neither appears anywhere in either file.

**Why it matters:** If someone repeatedly tries different tokens or
tries to access resources they're not allowed to, there's currently no
way to notice this happening, since nothing is written to any log.

**Suggested fix:** Add a simple log line (even just `console.warn`) when
a request is rejected with `401` or `403`, including the route and a
timestamp.

---

## What Was Done Correctly (Worth Noting)

Not everything in this review was a problem — a few things are worth
recognizing:

- **SQL Injection:** every database query seen in `server.js` uses
  parameterized queries (`$1`, `$2`, etc.) instead of building SQL
  strings manually. This correctly prevents SQL injection.
- **Secrets handling:** `db.js` pulls the database credentials from
  environment variables rather than hardcoding them, and `.gitignore`
  correctly excludes `.env` and `node_modules/` from version control.
- **Ownership checks done right elsewhere:** opening and closing an
  attendance session both correctly verify that the requesting staff
  member is the one assigned to that timetable/session before allowing
  the action — this is the same kind of check that is missing in
  Finding 1, so the pattern to fix it already exists in the codebase.
- **Student data scoping:** `/api/v1/students/me/attendance` correctly
  limits results to the logged-in student's own ID.

---

## Summary Table

| # | Finding | Severity | Confirmed By |
|---|---------|----------|--------------|
| 1 | No role/ownership check on section student list | Critical | Full code read of route handler |
| 2 | `dotenv` pinned to a non-existent version | High | Checked against current npm registry |
| 3 | JWT algorithm not restricted in `jwt.verify()` | Medium | Direct code read |
| 4 | Role taken from token, not re-checked against DB | Medium | Full code read of `authorizeRole` |
| 5 | No logging of failed auth/permission attempts | Low | Searched both files for logging code |

## Priority Order for Fixes

1. Finding 1 — fix first, it's an active data exposure.
2. Finding 2 — fix before the next install on any machine.
3. Finding 3 — quick, low-effort fix.
4. Finding 4 — document as a known tradeoff, revisit if token lifetime changes.
5. Finding 5 — add basic logging when time allows.
