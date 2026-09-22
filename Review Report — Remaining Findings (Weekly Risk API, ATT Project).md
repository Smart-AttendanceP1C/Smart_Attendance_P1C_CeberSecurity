# Security Review Report — Remaining Findings (Weekly Risk API, ATT Project)

## 1. Purpose and Context

This report covers `weekly_risk_api.py`, the FastAPI service that wraps
the logistic regression model used to predict a student's next-week
attendance risk. The file already contains fixes for six previously
identified issues (referenced in the code as "Security Finding 1–6").
This report documents **additional findings that remain unresolved** in
the current version of the file.

---

## 2. Confirmed Findings

### Finding 1 — Insecure Default Value for SERVICE_KEY

**Code:**
```python
SERVICE_KEY = os.environ.get("AI_SERVICE_KEY", "local-dev-only-change-me")
```

**Description:** If the `AI_SERVICE_KEY` environment variable is not
set, the service silently falls back to a hardcoded default value that
is visible to anyone who reads the source code.

**How this was verified:** Read the line directly — the fallback string
is written in plain text in the source file itself, and the comment
above it confirms this is intended only for local testing ("Set it as
an environment variable in real deployment — never hardcode the real
value").

**Impact:** If the environment variable is left unset in a real
deployment (e.g., due to a missed configuration step), the service
runs with a publicly known key. Anyone who has read the source — which
includes anyone with access to this repository — could authenticate as
the backend by sending `X-Service-Key: local-dev-only-change-me`,
bypassing `require_service_key` entirely.

**Recommendation:** Fail startup (raise an exception) if
`AI_SERVICE_KEY` is not set, instead of silently falling back to a
known default.

---

### Finding 2 — Fully Open CORS Policy

**Code:**
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Description:** The CORS policy allows requests from any origin,
using any method, with any headers.

**How this was verified:** Read the middleware configuration directly.
The accompanying comment acknowledges this is intended for local
testing only and states it "would be locked down to the actual
backend's origin" in real integration — but the code itself does not
enforce that restriction.

**Impact:** Any website loaded in a user's browser can issue
cross-origin requests to this API. Combined with any future issue that
leaks or guesses the service key from client-side code, this widens
the attack surface unnecessarily.

**Recommendation:** Restrict `allow_origins` to the specific origin(s)
of the real backend before deployment, and avoid `"*"` outside of local
development.

---

### Finding 3 — Residual Risk from Pickle-Based Model Loading

**Code:**
```python
model = joblib.load(MODEL_PATH)
```

**Description:** The integrity check (`verify_model_integrity`)
mitigates silent tampering after the hash is recorded, but does not
eliminate the underlying risk of `joblib`/`pickle` deserialization,
which can execute arbitrary code embedded in the file.

**How this was verified:** Read `verify_model_integrity()` and its own
docstring, which explicitly states: "This doesn't stop pickle's
arbitrary-code-execution risk on its own, but it stops a SILENTLY
swapped/tampered file from ever being loaded."

**Impact:** If an attacker can influence the model file *before* the
hash is recorded (e.g., during training or in the training pipeline),
or if both the model file and its hash manifest are compromised
together, this check provides no protection, and `joblib.load()` will
still execute whatever code the malicious file contains.

**Recommendation:** Treat the hash check as a partial mitigation only.
Consider a non-executable model serialization format (e.g., ONNX) for
a stronger long-term fix, and restrict write access to both the model
file and its hash manifest to the training pipeline alone.

---

### Finding 4 — Unsanitized Exception Details in Logs

**Code:**
```python
logger.warning("Prediction input rejected by model (%s); using fallback.", exc)
```

**Description:** Exception messages from the model layer are logged
in full, without filtering or redaction.

**How this was verified:** Read both exception-handling branches in
`predict_weekly_risk()` and confirmed neither filters or truncates the
exception content before logging it.

**Impact:** If an underlying library's exception message includes
details about input data or internal model structure, that
information is written verbatim to the log file, an output separate
from the API's own access controls. Anyone with log access — which may
be broader than the set of people authorized to query the API — could
gain information they would not otherwise be entitled to.

**Recommendation:** Log a generic, fixed message at the point the
exception is caught, and record the full exception detail only in a
separate, access-controlled debug channel if needed.

---

### Finding 5 — No Request Size Limit

**Description:** Neither FastAPI/Starlette nor this service defines a
maximum request body size for `/predict-weekly-risk`.

**How this was verified:** Reviewed the full file for any body-size
middleware or configuration — none is present. Pydantic validation
only runs after the body has already been read and parsed.

**Impact:** A client can send an arbitrarily large request body. Even
though Pydantic will eventually reject invalid field values, the
service still spends CPU and memory parsing the oversized payload
first, which is a low-effort denial-of-service vector if repeated at
volume.

**Recommendation:** Add explicit request body size limits at the
web-server or middleware level.

---

## 3. Summary Table

| # | Finding | Verified By |
|---|---------|-------------|
| 1 | Insecure default value for `SERVICE_KEY` | Read source line; fallback value is hardcoded and visible to anyone with repo access |
| 2 | Fully open CORS policy (`allow_origins=["*"]`) | Read middleware config; comment confirms it is unrestricted |
| 3 | Residual pickle deserialization risk in model loading | Read `verify_model_integrity()` docstring, which states the risk is not fully eliminated |
| 4 | Unsanitized exception details written to logs | Read both exception branches in `predict_weekly_risk()` |
| 5 | No request body size limit | Reviewed full file for size-limiting middleware — none found |
