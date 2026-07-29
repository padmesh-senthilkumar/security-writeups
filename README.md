# Web Application Security Assessment

**Target:** DVWA (Damn Vulnerable Web Application)
**Prepared by:** Padmesh S
**Environment:** Kali Linux (VirtualBox) — DVWA via Docker

---

## Executive Summary

Over a multi-week period, a security assessment was conducted against DVWA (Damn Vulnerable Web Application), a legally sanctioned platform for practicing web application security testing. The assessment identified three critical vulnerability categories: SQL Injection, Cross-Site Scripting (XSS), and Command Injection. Each vulnerability was successfully exploited in a controlled environment to demonstrate real-world impact, including unauthorized data extraction, session hijacking potential, and arbitrary command execution on the host system. Findings and remediation recommendations are detailed below.

## Scope & Methodology

Testing was performed against a locally hosted instance of DVWA running at security level **"low,"** inside an isolated Kali Linux virtual machine (VirtualBox, NAT networking). All testing was conducted exclusively against the tester's own environment, in accordance with responsible and legal security research practices. Tools used included the built-in browser, the Linux terminal, Hydra, John the Ripper, and manual payload crafting.

## Findings Summary

| # | Finding | Severity |
|---|---------|----------|
| 1 | SQL Injection | High |
| 2 | Cross-Site Scripting (Reflected XSS) | Medium-High |
| 3 | Command Injection | High |

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

**Impact:** An attacker could execute arbitrary commands on the server read sensitive files, create backdoor accounts, deploy malware, or pivot to attack other systems on the network. This represents full compromise potential, not just data exposure.

**Recommendation:** Avoid passing user input directly to system shell commands. Use safe API calls or built-in language functions instead of shell execution wherever possible. Where shell commands are unavoidable, strictly whitelist allowed input (e.g., valid IP address formats only) and reject or escape special characters such as `;`, `&&`, and `|`.

---

## Conclusion

The assessment of DVWA revealed a consistent underlying pattern across all three findings: a failure to properly separate user-supplied input from executable code or commands. Whether in SQL queries, HTML/JavaScript rendering, or system shell execution, each vulnerability stemmed from the application trusting user input without adequate validation or sanitization. Given the severity of these findings ranging from full database disclosure to complete server compromise the overall risk profile of this application, if deployed in production, would be classified as **Critical**. It is strongly recommended that input validation, output encoding, and parameterized/prepared statements be treated as baseline requirements in the development lifecycle, not optional safeguards.
