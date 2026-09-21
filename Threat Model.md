# Threat Model
## Smart Attendance & Student Presence Verification System

**Status:** Preliminary Analysis — Version 1.0
**Date:** 21 September 2026

---

## 1. Overview

This document presents an initial threat model for the Smart Attendance &
Student Presence Verification System.

The system includes four main roles: Student, Lecturer/TA, Administrator,
and Auditor.

The main security areas are authentication, authorization, attendance, QR
codes, location validation, file uploads, APIs, and data protection.

This is a preliminary analysis. The identified threats will be verified
during the security testing phase.

---

## 2. Important Assets

| Asset | Security Concern |
|---|---|
| Student Data | Unauthorized access or disclosure |
| User Credentials | Account takeover |
| Attendance Records | Unauthorized modification |
| QR Tokens | Replay or manipulation |
| Session Data | Unauthorized attendance |
| Audit Logs | Modification or deletion |
| Evidence Files | Unsafe or unauthorized files |
| CSV Files | Malicious or invalid input |
| Reports | Sensitive data exposure |
| Location Data | Attendance verification bypass |

---

## 3. Threat Actors

Potential threat actors include:

- Unauthenticated attackers
- Malicious students
- Compromised user accounts
- Malicious insiders
- Automated attackers

---

## 4. Main Threats

| ID | Threat | Possible Impact | Main Control |
|---|---|---|---|
| T01 | Broken Access Control | Access to another student's data | Backend authorization |
| T02 | Privilege Escalation | Student accesses Admin/TA functions | RBAC |
| T03 | Account Takeover | Unauthorized account access | Secure authentication + MFA |
| T04 | QR Replay | Fake/duplicate attendance | Expiration + rotation |
| T05 | QR Tampering | Manipulated attendance request | Cryptographic signature |
| T06 | Attendance Manipulation | Incorrect attendance records | Server-side validation |
| T07 | Geofence Bypass | Attendance from unauthorized location | Server-side geofence validation |
| T08 | Malicious CSV Upload | Data corruption or abuse | Validation + sanitization |
| T09 | Unsafe File Upload | Malicious/unauthorized files | File validation + access control |
| T10 | Data Exposure | Leakage of student information | RBAC + response filtering |
| T11 | Audit Log Tampering | Loss of security evidence | Protected audit logs |
| T12 | API Abuse | Service degradation | Rate limiting |

---

## 5. STRIDE Classification

| Category | Example |
|---|---|
| Spoofing | Account takeover |
| Tampering | Attendance or QR modification |
| Repudiation | Denying a security-sensitive action |
| Information Disclosure | Unauthorized student data access |
| Denial of Service | Excessive API requests |
| Elevation of Privilege | Student accessing Admin functionality |

---

## 6. Initial Risk Assessment

| Threat | Likelihood | Impact | Risk |
|---|---|---|---|
| Broken Access Control | High | High | High |
| Privilege Escalation | Medium | Critical | High |
| Account Takeover | Medium | High | High |
| QR Replay | Medium | High | High |
| QR Tampering | Medium | High | High |
| Attendance Manipulation | Medium | High | High |
| Geofence Bypass | Medium | High | High |
| File / CSV Upload | Medium | High | High |
| Data Exposure | Medium | High | High |
| API Abuse | Medium | High | High |

Risk levels are preliminary and will be updated after actual security
testing.

---

## 7. Security Controls to Verify

The following controls should be verified during testing:

- Backend authentication and authorization
- RBAC enforcement on protected endpoints
- Object-level access control
- QR signature and expiration validation
- Replay protection
- Attendance uniqueness
- Server-side geofence validation
- File and CSV validation
- Secure handling of sensitive data
- Protected audit logging
- API rate limiting

---

## 8. Testing Status

The threats in this document are currently considered potential risks and
have not yet been confirmed as vulnerabilities.

The next step is to test the backend APIs and application functionality
and document any confirmed vulnerabilities separately.

---

## 9. Conclusion

The main security concerns for the system are access control,
authentication, attendance integrity, QR security, location validation,
file handling, API security, and protection of student data.

The threat model will be updated as security testing progresses and new
findings are identified.
