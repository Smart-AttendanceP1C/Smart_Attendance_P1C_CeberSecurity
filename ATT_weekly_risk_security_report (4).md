# Security Review Report — Weekly Attendance Risk Module (ATT Project)

**Component:** AI-based weekly attendance risk prediction (ATT-FR-10)
**Files reviewed:** `train_weekly_risk_model.py`, `weekly_risk_api.py`, `weekly_risk_model.joblib`
**Reviewer:** Cybersecurity Team Member — ATT Project
**Date:** 21 September 2026

## 1. Purpose and Context

This module implements the AI risk-prediction feature required by the
project brief (ATT-FR-10 / US-18): a logistic regression model that
predicts which students are likely to have low attendance the following
week, so lecturers and administrators can follow up early.

Because this feature processes real attendance data belonging to real
students, any weakness in how it is secured is not an isolated technical
bug — it is a direct risk to the same student data the rest of the ATT
system is designed to protect (per ATT-FR-01, which requires role-based
access control on every backend endpoint).

This report lists every vulnerability found during the review, why it
matters specifically for this system, and the exact steps used to confirm
each one.

---

## 2. Findings

### Finding 1 — No Authentication on the Prediction Endpoint
**Severity:** Critical
**Location:** `weekly_risk_api.py`, `POST /predict-weekly-risk`

**Description:** The endpoint does not verify the caller's identity or
role before processing a request. Anyone who can reach the service over
the network can request a risk prediction, with no login and no token.

**Relevance to the project:** This directly violates the project's own
access-control requirement (ATT-FR-01) and its acceptance criteria that a
student must not be able to access another student's attendance data.
Since risk predictions are derived from real student attendance patterns,
this is effectively an unrestricted data-exposure path.

**How it was tested:** Sent the sample request documented in the file
itself with no Authorization header:
```bash
curl -i -X POST http://localhost:8000/predict-weekly-risk \
     -H "Content-Type: application/json" \
     -d '{"attendance_rate_to_date": 0.65, "trend_last_3": -0.1,
          "rejected_last_2": 1, "consecutive_absences": 2, "course_load": 5}'
```
**Result:** Returned `200 OK` with a full prediction. A secured endpoint
should have returned `401 Unauthorized`.

---

### Finding 2 — Unsafe Model Loading (Pickle Deserialization Risk)
**Severity:** Critical
**Location:** `weekly_risk_api.py` (`joblib.load(MODEL_PATH)`),
`train_weekly_risk_model.py` (`joblib.dump(model, MODEL_PATH)`)

**Description:** `joblib` serializes models using Python's `pickle`
format. If the model file on disk is replaced with a malicious one, the
API will execute arbitrary code the next time it loads it — with no
integrity check in place.

**Relevance to the project:** A compromised model file could silently
alter every risk prediction (falsely flagging or clearing students) or
serve as a foothold to compromise the server hosting other parts of the
attendance system.

**How it was tested:** Checked file permissions on the deployed model
file (`ls -la weekly_risk_model.joblib`) and found write access is not
restricted to a single trusted account. Confirmed by code review that no
hash check, signature, or other verification exists before the file is
loaded. No malicious payload was created or executed — the missing
integrity check and the permission finding are sufficient to confirm the
risk without needing to weaponize it.

---

### Finding 3 — No Rate Limiting
**Severity:** Medium
**Location:** `weekly_risk_api.py`, `POST /predict-weekly-risk`

**Description:** There is no limit on how many requests a single client
can send.

**Relevance to the project:** Combined with Finding 1 (no auth), this
allows an outside party to flood the service (denial of service) or
systematically vary inputs to infer how the model scores students —
effectively reverse-engineering the risk logic used on real students.

**How it was tested:**
```bash
for i in {1..200}; do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8000/predict-weekly-risk \
    -H "Content-Type: application/json" \
    -d '{"attendance_rate_to_date": 0.65, "trend_last_3": -0.1,
         "rejected_last_2": 1, "consecutive_absences": 2, "course_load": 5}'
done | sort | uniq -c
```
**Result:** All 200 requests returned `200 OK` with no throttling or
`429 Too Many Requests` response.

---

### Finding 4 — Information Disclosure via `/health`
**Severity:** Low
**Location:** `weekly_risk_api.py`, `GET /health`

**Description:** This public, unauthenticated endpoint reveals whether
the real ML model is loaded or the system has fallen back to the simpler
rule-based method.

**Relevance to the project:** On its own this is not a data leak, but it
functions as reconnaissance: it tells an outside party exactly when the
system is running in its weaker, more predictable fallback mode (see
Finding 5), which could inform the timing of an attack against other
weaknesses such as Finding 1.

**How it was tested:** Called `/health` with no authentication — returned
the status normally. Then temporarily renamed the model file, restarted
the service, and called `/health` again. The response correctly changed
to reflect `model_loaded: false`, confirming the endpoint accurately and
reliably exposes this internal state to anyone.

---

### Finding 5 — Predictable Fallback Response Values
**Severity:** Low
**Location:** `weekly_risk_api.py`, `baseline_fallback()`

**Description:** When the model is unavailable, the API always returns
one of exactly two fixed probabilities: `0.8` (flagged) or `0.1` (not
flagged), regardless of the specific input values.

**Relevance to the project:** Combined with Finding 4, this means the
exact output becomes guessable once a caller knows the system is in
fallback mode — undermining the reliability of the risk signal that
lecturers/administrators are meant to trust.

**How it was tested:** With the model file removed, sent several
different prediction requests; every response returned exactly `0.8` or
`0.1`, with `source: "baseline_fallback"`.

---

### Finding 6 — Overly Broad Exception Handling
**Severity:** Low
**Location:** Both `weekly_risk_api.py` and `train_weekly_risk_model.py`
(`except Exception as exc`)

**Description:** All failures — a benign input mistake, a corrupted
model, or an actual exploitation attempt — are caught identically and
logged as a generic warning.

**Relevance to the project:** This makes it difficult to notice if
someone is actively probing this part of the attendance system, since
routine errors and attack attempts look the same in the logs.

**How it was tested:** Sent a structurally valid but extreme value
(`consecutive_absences: 999999999`) and observed the server logs — only a
single generic warning line was produced, with no distinguishing detail
or severity level.

---

## 3. Summary Table

| # | Finding | Severity | Confirmed |
|---|---------|----------|-----------|
| 1 | No authentication/authorization on `/predict-weekly-risk` | Critical | Yes |
| 2 | Unsafe pickle-based model loading, no integrity check | Critical | Yes |
| 3 | No rate limiting | Medium | Yes |
| 4 | `/health` discloses internal model state, unauthenticated | Low | Yes |
| 5 | Fixed, guessable fallback probabilities | Low | Yes |
| 6 | Broad exception handling hides attack signals | Low | Yes |

---

## 4. Recommendations and Priority

1. **Finding 1 (Critical):** Add role-based authentication to
   `/predict-weekly-risk`, consistent with ATT-FR-01's requirement that
   every backend endpoint enforce role checks server-side.
2. **Finding 2 (Critical):** Restrict write access to
   `weekly_risk_model.joblib` to a single trusted deployment account, and
   add integrity verification (hash/signature check) before loading it.
3. **Finding 3 (Medium):** Add rate limiting to the prediction endpoint.
4. **Findings 4–6 (Low):** Simplify `/health` to a generic status
   response, move detailed status behind an authenticated admin
   endpoint, and add structured/leveled logging to separate routine
   errors from suspicious activity.

**Testing scope note:** All tests were performed against a local/staging
instance of the service. No test was run against a production
environment, and no malicious payload was crafted or executed at any
point in this review.
