# Insecure Storage of Authentication Data

## 1. Vulnerability Overview

During the security assessment of the Smart Attendance mobile application, a vulnerability was identified in the way authentication-related information is stored locally.

The application stores authentication and user-related information inside Flutter `SharedPreferences`, including an authentication token, user ID, user role, and email.

The identified entries included:

```text
flutter.auth_token
flutter.user_id
flutter.user_role
flutter.user_email
flutter.user_name
```

The authentication token was found under:

```text
flutter.auth_token
```

This represents an **Insecure Local Storage** security issue because authentication-related information is being persisted in general application preferences instead of a dedicated secure-storage mechanism.

---

# 2. How the Vulnerability Was Discovered

The assessment started by examining the Android application and identifying its package:

```text
com.example.verishift_app
```

The application was then connected to an authorized test Android device through ADB.

The package configuration was inspected and the application was found to have the `DEBUGGABLE` flag enabled.

The application's debugging context was then used to inspect its private test storage.

The Flutter SharedPreferences file was identified as:

```text
FlutterSharedPreferences.xml
```

The file contained authentication-related information such as:

```text
flutter.user_role=student
flutter.user_email=student.demo@bua.edu.eg
flutter.user_name=Salma Mahmoud
flutter.auth_token=demo_token
flutter.user_id=2023000000
```

The important finding was the presence of the authentication token in the application's local preferences.

---

# 3. Why This Is a Vulnerability

Authentication tokens are security-sensitive because they represent the authenticated session of a user.

When a valid token is exposed, an attacker who obtains it may be able to send requests to protected API endpoints using the token.

For example, a protected API normally expects:

```http
Authorization: Bearer <access_token>
```

If an attacker obtains a valid token, the server may recognize the request as coming from the legitimate authenticated user.

Therefore, exposing the token can potentially bypass the need to know the user's password.

The important point is that the server may trust the token as proof of authentication.

---

# 4. Potential Attack Scenario

A realistic attack scenario would be:

**Step 1:** An attacker obtains access to the application's local data.

**Step 2:** The attacker discovers the locally stored authentication token.

**Step 3:** The attacker extracts the token.

**Step 4:** If the token is valid, the attacker could potentially send it with requests to protected API endpoints.

**Step 5:** The API validates the token and may treat the attacker as the legitimate user.

The resulting access would depend on the permissions and role associated with the compromised token.

For example, a student token would only provide the permissions assigned to that student account, while a token belonging to a privileged account could have a much greater impact.

---

# 5. Risk / Impact

The main security risk is **Authentication Token Exposure**.

Potential consequences include:

* Unauthorized access to protected API resources.
* Session hijacking if a valid token is obtained.
* Exposure of user information.
* Unauthorized actions within the permissions of the compromised account.
* Greater impact if a privileged account token is exposed.

The risk becomes significantly higher when the application stores long-lived or highly privileged authentication tokens.

---

# 6. Evidence From the Assessment

The following authentication-related value was observed:

```text
flutter.auth_token=demo_token
```

Additional user information was also stored:

```text
flutter.user_id=2023000000
flutter.user_role=student
flutter.user_email=student.demo@bua.edu.eg
```

The application was also confirmed to be running with debugging enabled.

These observations provide evidence that authentication-related information is being persisted locally.

---

# 7. Important Limitation

The discovered token was:

```text
demo_token
```

Therefore, the assessment did not claim that this specific value could be used to take over a real account.

The vulnerability is the **insecure storage design**, while the actual impact depends on whether a real and valid authentication token can be recovered and abused.

---

# 8. Severity

The severity should be determined based on the actual production environment.

If a production application stores a valid authentication token in an insecure location and that token can be recovered by an attacker, the issue could have a significant impact because possession of the token may provide authenticated access.

For the tested build, the finding is documented as an **Insecure Local Storage of Authentication Data** issue because the discovered token was a demo value.

---

# 9. Recommended Fix

The application should not rely on normal `SharedPreferences` for sensitive authentication credentials.

Instead:

* Store authentication tokens using Android secure storage mechanisms.
* Use Flutter secure-storage solutions backed by Android Keystore where appropriate.
* Avoid storing sensitive information unnecessarily.
* Make production builds non-debuggable.
* Review Android backup configuration.
* Use appropriate token expiration and revocation.
* Use short-lived access tokens where appropriate.
* Maintain server-side authorization even if a token is compromised.

---

# 10. Verification After Remediation

After fixing the issue, the security team should reinstall the production/release APK and verify that:

1. The application is no longer debuggable.
2. Authentication tokens are not stored in ordinary SharedPreferences.
3. Sensitive tokens are stored using secure storage.
4. Application backups do not expose sensitive authentication data.
5. Token expiration and revocation work correctly.
6. Protected APIs continue enforcing authorization independently of the client.

---

# 11. Short Explanation for the Viva

If the professor asks:

**"What vulnerability did you find?"**

Answer:

> "I identified an insecure local storage issue in the mobile application. During the Android security assessment, I inspected the application's local storage and found authentication-related data, including the auth token, inside Flutter SharedPreferences. The problem is that authentication tokens are sensitive credentials, and if a valid token is exposed, it could potentially be reused to access protected API endpoints as the affected user. In our test, the discovered token was a demo value, so we did not claim an actual account takeover. Our recommendation is to use secure storage backed by Android Keystore and disable debugging in production."

## One sentence to remember

> **"The vulnerability is not simply that user data exists locally; the security issue is that authentication-related data, especially the token, is stored in an insufficiently protected storage mechanism."**
