# Web Application Security Assessment

**Target:** DVWA (Damn Vulnerable Web Application)
**Prepared by:** Padmesh S
**Environment:** Kali Linux (VirtualBox) — DVWA via Docker

---

## Executive Summary

Over a multi-week period, a security assessment was conducted against DVWA (Damn Vulnerable Web Application), a legally sanctioned platform for practicing web application security testing. The assessment identified six vulnerability categories: SQL Injection, Cross-Site Scripting (XSS), Command Injection, Unrestricted File Upload, Cross-Site Request Forgery (CSRF), and Weak Session IDs. Each vulnerability was successfully exploited in a controlled environment to demonstrate real-world impact, including unauthorized data extraction, session hijacking, silent account takeover, and full remote code execution on the host system. Findings and remediation recommendations are detailed below.

## Scope & Methodology

Testing was performed against a locally hosted instance of DVWA running at security level **"low,"** inside an isolated Kali Linux virtual machine (VirtualBox, NAT networking). All testing was conducted exclusively against the tester's own environment, in accordance with responsible and legal security research practices. Tools used included the built-in browser, the Linux terminal, Firefox Developer Tools, Hydra, John the Ripper, and manual payload crafting.

## Findings Summary

| # | Finding | Severity |
|---|---------|----------|
| 1 | SQL Injection | High |
| 2 | Cross-Site Scripting (Reflected XSS) | Medium-High |
| 3 | Command Injection | High |
| 4 | Unrestricted File Upload | High |
| 5 | Cross-Site Request Forgery (CSRF) | High |
| 6 | Weak / Predictable Session IDs | High |

---

## Finding 1: SQL Injection

**Severity:** High

**Description:** The application fails to sanitize user input before inserting it into a SQL query, allowing an attacker to inject arbitrary SQL logic.

**Steps to Reproduce:**

```sql
' OR '1'='1
```

Entering the payload above into the User ID field caused the query to return all records instead of the single intended record. A follow-up UNION-based payload was used to extract usernames and password hashes directly:

```sql
' UNION SELECT user, password FROM users -- -
```

**Impact:** Full disclosure of the `users` table (usernames and password hashes), enabling further attacks such as offline password cracking.

**Recommendation:** Implement parameterized queries (prepared statements) so user input is always treated strictly as data, never as executable SQL logic. Input validation and least-privilege database accounts should be used as additional layers of defense.

---

## Finding 2: Cross-Site Scripting (Reflected XSS)

**Severity:** Medium-High

**Description:** The application reflects user input directly into the page without sanitization, allowing injected JavaScript to execute in the victim's browser.

**Steps to Reproduce:**

```html
<script>alert('XSS')</script>
```

Submitting the payload above in the input field caused the script to execute upon page load, confirmed via a JavaScript alert dialog.

**Impact:** An attacker could hijack a victim's session cookie and impersonate them. If the victim holds administrative privileges, the attacker gains equivalent access within the application.

**Recommendation:** Implement output encoding so user input is always rendered as plain text, never executable code. Deploy a Content Security Policy (CSP) as a secondary defense layer.

---

## Finding 3: Command Injection

**Severity:** High

**Description:** The application passes user input directly into a system shell command (used for a ping function) without sanitization, allowing an attacker to chain additional commands using characters such as `;` or `&&`.

**Steps to Reproduce:**

```bash
127.0.0.1; whoami
```

Entering the payload above into the ping input field executed the intended ping command followed by an unauthorized `whoami` command, confirming arbitrary command execution on the host.

**Impact:** An attacker could execute arbitrary commands on the server — read sensitive files, create backdoor accounts, deploy malware, or pivot to attack other systems on the network. This represents full compromise potential, not just data exposure.

**Recommendation:** Avoid passing user input directly to system shell commands. Use safe API calls or built-in language functions instead of shell execution wherever possible. Where shell commands are unavoidable, strictly whitelist allowed input (e.g., valid IP address formats only) and reject or escape special characters such as `;`, `&&`, and `|`.

---

## Finding 4: Unrestricted File Upload

**Severity:** High

**Description:** The application's file upload feature does not validate the type or content of uploaded files, allowing an attacker to upload and execute a malicious script.

**Steps to Reproduce:**

```php
<?php system($_GET["cmd"]); ?>
```

Saved as `shell.php` and uploaded via the File Upload form. The file uploaded successfully with no restriction on file type. Navigating to the uploaded file's path and appending a command parameter confirmed remote code execution:

```
http://127.0.0.1/hackable/uploads/shell.php?cmd=whoami
```

The server executed the command and returned `www-data` directly in the browser.

**Impact:** An attacker gains a persistent web shell on the server, enabling arbitrary and repeatable command execution at will, without needing to re-exploit the application. This represents a more severe and durable compromise than a one-off command injection.

**Recommendation:** Whitelist allowed file extensions and reject all others. Validate actual file content/magic bytes rather than trusting the filename or declared MIME type. Store uploaded files outside the web-executable directory, or disable script execution within the upload directory. Rename uploaded files server-side to remove attacker control over the filename.

---

## Finding 5: Cross-Site Request Forgery (CSRF)

**Severity:** High

**Description:** The password change function accepts sensitive state-changing requests via GET parameters with no verification that the request was intentionally submitted by the authenticated user, and no CSRF token protection.

**Steps to Reproduce:**

```
http://127.0.0.1/vulnerabilities/csrf/?password_new=hacked123&password_conf=hacked123&Change=Change
```

Simply loading this URL while authenticated silently changed the logged-in user's password, with no confirmation step and no interaction with the actual form.

**Impact:** An attacker can craft a malicious link or embed the request (e.g., as a hidden image tag) on an external site. If a victim with an active session loads it, their password is silently changed without their knowledge, resulting in full account takeover.

**Recommendation:** Use POST requests for all state-changing actions rather than GET. Implement unique, unpredictable CSRF tokens embedded in each form and validate them server-side on submission. Consider requiring re-authentication (current password) for sensitive actions such as password changes.

---

## Finding 6: Weak / Predictable Session IDs

**Severity:** High

**Description:** Session identifiers issued by the application are generated as a simple incrementing counter rather than a cryptographically random value.

**Steps to Reproduce:**

Repeatedly triggering session generation via the application returned sequential, predictable values (e.g., `50, 51, 52...`) for the `dvwaSession` cookie.

**Impact:** An attacker who observes their own session ID can predict or brute-force nearby values to hijack another user's active session without needing their password, credentials, or any other interaction with the victim.

**Recommendation:** Generate session identifiers using a cryptographically secure random number generator, producing long (32+ character), high-entropy values that cannot be feasibly guessed or enumerated. Rotate session IDs on privilege changes (e.g., login) and enforce expiration.

---

## Conclusion

The assessment of DVWA revealed a consistent underlying pattern across the majority of findings: a failure to properly separate user-supplied input from executable code or commands. This was evident in SQL Injection, XSS, Command Injection, and Unrestricted File Upload alike each stemmed from the application trusting user input (whether typed, scripted, or uploaded) without adequate validation or sanitization. The remaining two findings, CSRF and Weak Session IDs, point to a second, related theme: insufficient protection of the session and request-authenticity layer, allowing an attacker to act as, or impersonate, a legitimate user without ever needing their credentials. Given the severity of these findings ranging from full database disclosure to complete server compromise and silent account takeover the overall risk profile of this application, if deployed in production, would be classified as **Critical**. It is strongly recommended that input validation, output encoding, parameterized/prepared statements, CSRF tokens, and cryptographically random session identifiers be treated as baseline requirements in the development lifecycle, not optional safeguards.
