1. Purpose and Context

This file defines the documented contract for the Smart Attendance backend API — every endpoint, its required security, and its expected responses. Reviewing it ahead of the backend code itself helps determine whether access-control gaps are a documentation oversight or a sign of a deeper issue that needs to be verified in the actual implementation.

2. Findings
Finding 1 — No Documented Protection Against Brute-Force Login Attempts

Location: /api/v1/auth/login

yaml
responses:
  '200': Login successful
  '401': Invalid email or password
  '400': Validation error

Description: Only three outcomes are documented for the login endpoint. There is no 429 Too Many Requests response, and no mention of rate limiting or account lockout anywhere in the file.

Verification method: Searched the entire document (not just this endpoint) for the string 429 and the terms rate limit, lockout, and too many — none appear anywhere. This confirms the absence is document-wide rather than specific to one endpoint.

Impact: As written, the spec gives no indication the system limits repeated login attempts, which would allow unrestricted password guessing. This needs to be checked against the actual backend code — the protection may exist in the implementation but simply go undocumented, or it may genuinely be missing.

Finding 2 — Realistic-Looking Password Example in the Spec

Location: LoginRequest schema

yaml
password:
  type: string
  format: password
  example: Password123!

Description: The example value resembles a real, usable password rather than an obvious placeholder (e.g. <your_password>).

Verification method: Directly visible in the source — the value is written literally in the schema definition.

Impact: If this file is shared publicly, the example reveals the expected password-policy shape, and in practice teams sometimes reuse example credentials as real test accounts.

Finding 3 — /sections/{sectionId}/students Has No Documented Role Check

Location: /api/v1/sections/{sectionId}/students

yaml
responses:
  '200': ...
  '401': ...
  '404': ...

Description: Only three outcomes are documented: success, not authenticated, or section not found. There is no 403 response documenting the case where an authenticated user isn't authorized to view a specific section's students.

Verification method: Compared this endpoint against another in the same file, /api/v1/students/me/attendance, which explicitly documents:

yaml
'403':
  description: Student access required

This shows the spec's authors do document role checks elsewhere, making the absence here a direct, confirmed inconsistency rather than a general document-wide omission.

Impact: The project brief (ATT-FR-03) requires that a lecturer/TA only see sections they are responsible for. As documented, this endpoint does not reflect that restriction. This is the highest-priority item to verify against the actual backend code, since if the implementation matches the spec exactly, it would confirm a Broken Access Control vulnerability.

Finding 4 — /courses and /timetable Have No Documented Role Distinction

Location: /api/v1/courses, /api/v1/timetable

yaml
responses:
  '200': ...
  '401': ...

Description: Both endpoints require only authentication (401), with no 403 distinguishing between roles.

Verification method: Same endpoint-by-endpoint comparison method as Finding 3.

Impact: This may be intentional (courses and timetables could be shared, non-sensitive data visible to all authenticated users) or an oversight. This needs confirmation from the development team rather than being treated as a confirmed vulnerability.

Finding 5 — No Response Schema Defined for Successful Responses

Location: All 200/201 responses throughout the file

yaml
'200':
  description: Users retrieved successfully

Description: No response includes a content:/schema: block describing the fields actually returned.

Verification method: Searched the entire file for the string content: under any 200 or 201 response block — it does not appear anywhere.

Impact: Without a defined response schema, it isn't possible to confirm from this document alone whether endpoints like /api/v1/users return only necessary fields or also expose sensitive data (e.g. password hashes, national IDs). This is a documentation gap that limits how much can be reviewed without the backend source code.

3. Summary Table
#	Finding	Verified By
1	No documented brute-force protection on /login	Document-wide search for 429/rate-limit terms — none found
2	Realistic password example in spec	Directly visible in the LoginRequest schema
3	/sections/{id}/students missing documented 403	Direct comparison with /students/me/attendance, which has one
4	/courses and /timetable missing documented 403	Same endpoint comparison method
5	No response schema on any success response	Document-wide search for content: — none found
4. Next Steps
Finding 3 is the highest priority to verify against the actual backend implementation, since it maps directly to project requirement ATT-FR-03 and, if the code matches the spec, would confirm a real access-control vulnerability.
Findings 1 and 5 also require the backend source code to determine whether the gap is documentation-only or present in the actual system.
Finding 4 should be confirmed with the development team as either an intentional design decision or an oversight.
