# Security Testing Report – ATT Weekly Attendance Risk API

## Security Testing Activities

### 1. Authentication Testing

Tested the `/admin/model-status` endpoint using Burp Suite Repeater without providing the `x-service-key` header.

**Result:**
The server returned:

```text
401 Unauthorized
```

This confirms that the administrative endpoint requires authentication.

### 2. Service Key Authentication

Retested the same endpoint using the authorized `x-service-key` provided by the project API configuration.

**Result:**
The request was successfully authenticated and the endpoint returned the model status.

### 3. API Documentation Review

Reviewed the Swagger/OpenAPI documentation and identified that the Service Key was included directly in the generated cURL example.

**Security concern:**
If the API documentation is accessible to unauthorized users, exposing the Service Key could allow unauthorized access to protected endpoints.

**Recommendation:**
Use placeholders instead of real secrets in API documentation and restrict access to Swagger/OpenAPI documentation in production.

### 4. Input Validation Testing

Reviewed the `/predict-weekly-risk` endpoint and its input parameters:

* `attendance_rate_to_date`
* `trend_last_3`
* `rejected_last_2`
* `consecutive_absences`
* `course_load`

Test cases were considered for invalid, negative, out-of-range, and incorrect data types to verify proper API validation.

### 5. SQL Injection Assessment

Reviewed the available endpoints for parameters that could be used to test SQL Injection.

The `/admin/model-status` endpoint does not accept user-controlled parameters, and `/predict-weekly-risk` mainly accepts numerical prediction inputs.

**Result:**
No SQL Injection vulnerability was confirmed from the endpoints tested so far.

Further SQL Injection testing requires database-related endpoints such as user search, student lookup, course lookup, or attendance queries.

## Summary

The main security observation identified during the current testing phase was the exposure of a Service Key within the API documentation example. Authentication testing also confirmed that the administrative endpoint rejects requests without the required Service Key.
