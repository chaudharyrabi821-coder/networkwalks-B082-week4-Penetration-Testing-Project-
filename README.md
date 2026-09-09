# 🏥 Mediroza General Hospital — Black-Box Penetration Test

![Status](https://img.shields.io/badge/status-M1%20%26%20M2%20Complete-success)
![Type](https://img.shields.io/badge/engagement-Black--Box%20Pentest-blue)
![Scope](https://img.shields.io/badge/scope-medirozahospital.com-orange)
![Authorization](https://img.shields.io/badge/authorization-Written%20Permission%20Granted-brightgreen)
![Duration](https://img.shields.io/badge/duration-3%20days-lightgrey)

> **Program:** NetworkWalks — Batch B082, Week 4 &nbsp;|&nbsp; **Client (simulated):** Mediroza General Hospital
> **Target:** [`https://medirozahospital.com`](https://medirozahospital.com)
> **Note:** This is a training engagement carried out against a purpose-built lab target under NetworkWalks' Week 4 project brief, with written permission granted for testing as documented in the project scope. The techniques documented here must never be applied to any system without explicit written authorization.

<p align="center">
<img width="984" height="552" alt="image" src="https://github.com/user-attachments/assets/a4c05900-2353-4a2e-98a9-9181b8d02efd" />

</p>

---

## 📋 Project Brief

| | |
|---|---|
| **Project Type** | Penetration Testing & Vulnerability Assessment |
| **Client** | Mediroza General Hospital |
| **Target** | `https://medirozahospital.com` |
| **Scope** | Full black-box penetration test — identify vulnerabilities, exploit them to demonstrate real impact, and document all findings in a professional report |
| **Rules of Engagement** | Testing limited to the target domain only. No social engineering. No denial of service. No testing outside agreed scope |
| **Authorization** | Written authorization granted by the client for security testing of their web infrastructure |
| **Timeline** | 3 days |

---

## 🎯 Engagement Milestones

| # | Milestone | Objective | Status |
|---|---|---|---|
| **M1** | Initial Access | Attack the website and retrieve 3 confidential patient PDF lab reports | ✅ **Complete** |
| **M2** | Data Extraction | Crack the encryption on all 3 retrieved files | ✅ **Complete** |
| **M3** | Attack (Cracking) | Uncover staff salary and shareholder data | ✅ **Complete** |
| **M4** | Pentest Report | Deliver a professional penetration testing report | 🔜 Final phase |

---

## ⚡ Headline Findings

| # | Finding | Severity | Endpoint |
|---|---|---|---|
| 1 | **SQL Injection — Authentication Bypass** (`admin' --`) → full account takeover with no valid password | 🔴 Critical | `POST /patient/login.php` |
| 2 | **Weak / Trivial PDF Passwords** protecting confidential pathology reports — all 3 cracked via dictionary attack | 🟠 High | Downloaded patient reports |
| 3 | **Username Enumeration** — login form confirms whether a username exists before checking the password | 🟡 Medium | `POST /patient/login.php` |
| 4 | **Directory Listing Enabled** on `/staff/` and `/old/`, exposing filenames directly | 🟠 High | `GET /staff/`, `GET /old/` |
| 5 | **Exposed Historical Database Backup** — `/old/mediroza_db_backup_2019.sql` publicly downloadable, containing staff PII and shareholder records | 🔴 Critical | `GET /old/mediroza_db_backup_2019.sql` |

---

## 🧭 Methodology

Testing followed a standard black-box methodology, broadly aligned with the **PTES** and **OWASP Testing Guide**:

```
Reconnaissance → Mapping → Analysis → Exploitation → Data Recovery → Reporting
```

**Toolchain:** `curl` (fast CLI recon), Burp Suite Community Edition (proxy, Repeater, Intruder), a Chromium browser proxied through Burp, and a hash-extraction + dictionary-attack utility for the encrypted PDFs.

---

## 🪜 Full Walkthrough

### Milestone 1 — Initial Access

**Step 1 — Command-line reconnaissance.** Before opening a browser, `curl` was used to fingerprint the stack with minimal footprint:

<p align="center">
  <img width="1252" height="202" alt="image" src="https://github.com/user-attachments/assets/8f372ddc-0b95-4c03-a1db-c2f45c33d29d" />

</p>

`robots.txt` was checked next — and it handed over far more than expected, explicitly disallowing `/patient/`, `/staff/`, and `/old/`:

<p align="center">
  <img width="1261" height="181" alt="image2" src="https://github.com/user-attachments/assets/e981f38f-c23c-44e2-993e-58192e0fd2bb" />

</p>

This is effectively a self-authored map of the site's most sensitive areas, discovered before a single page was manually browsed. The `sitemap.xml` referenced at the bottom of `robots.txt` was checked too, but only listed the public marketing pages (`index`, `about`, `doctors`, `contact`) — confirming the sensitive paths were deliberately excluded from the "official" map rather than simply forgotten:

<p align="center">
  <img src="./evidence/images/18-recon-sitemap-xml.png" width="620" alt="sitemap.xml showing only public marketing pages">
</p>

**Step 2 — Browser reconnaissance.** The `/patient/login.php` path flagged by `robots.txt` was opened directly in a Burp-proxied browser:

<p align="center">
  <img src="./evidence/images/03-login-page-baseline.png" width="620" alt="Patient Portal login page baseline">
</p>

**Step 3 — Baseline login test (username enumeration found).** A plausible-but-invalid username was submitted to observe normal application behavior, producing an explicit **"Username not found"** message — a secondary finding on its own, since the application validates username existence before checking the password:

<p align="center">
  <img src="./evidence/images/04-login-username-not-found.png" width="620" alt="Username not found error message">
</p>

**Step 4 — Manual confirmation & exploitation.** A single `'` reproduced a SQL syntax anomaly, confirming unsanitized input reaching the database layer. The classic authentication-bypass payload was then submitted as the username, with any value as the password:

<p align="center">
  <img src="./evidence/images/05-login-admin-bypass-entered.png" width="620" alt="admin' -- authentication bypass payload entered">
</p>

```
Username: admin' --
Password: anything
```

**Step 5 — Impact: unauthorized data access.** The bypass succeeded, granting access to "My Reports" — three password-protected pathology reports:

<p align="center">
  <img src="./evidence/images/06-portal-reports-list.png" width="680" alt="My lab reports page showing 3 downloadable PDFs">
</p>

This closes out **Milestone 1**: proof of unauthorized access, plus the 3 target PDF files.

---

### Milestone 2 — Cracking the Encryption

Each of the 3 downloaded PDFs opened with a password prompt, exactly as advertised on the portal itself ("Your reports are password protected"). Each file was treated as an independent target — the brief's own hint warned not to assume one approach would fit all three.

**`patient_report_1.pdf` — Sipho Dlamini**

<p align="center">
  <img src="./evidence/images/07-pdf1-password-prompt.png" width="340" alt="Password prompt for patient_report_1.pdf">
</p>

A hash was extracted locally and run through a dictionary attack — cracked on the first attempt: **`123456`**.

<p align="center">
  <img src="./evidence/images/08-pdf1-report-dlamini.png" width="420" alt="Decrypted pathology report for Sipho Dlamini">
</p>

**`patient_report_2.pdf` — Priya Reddy**

<p align="center">
  <img src="./evidence/images/09-pdf2-password-prompt.png" width="340" alt="Password prompt for patient_report_2.pdf">
</p>

Cracked on the second attempt against the same built-in wordlist: **`password`**.

<p align="center">
  <img src="./evidence/images/10-pdf2-report-reddy.png" width="420" alt="Decrypted pathology report for Priya Reddy">
</p>

**`patient_report_3.pdf` — Emily Thompson**

<p align="center">
  <img src="./evidence/images/11-pdf3-password-prompt.png" width="340" alt="Password prompt for patient_report_3.pdf">
</p>

Confirmed the brief's warning was well-founded — this password was neither `123456` nor `password`, but still fell to the same built-in dictionary: **`!@#$%^&`**.

<p align="center">
  <img src="./evidence/images/12-pdf3-report-thompson.png" width="420" alt="Decrypted pathology report for Emily Thompson">
</p>

| File | Patient | Password | Attempts |
|---|---|---|---|
| `patient_report_1.pdf` | Sipho Dlamini | `123456` | 1st |
| `patient_report_2.pdf` | Priya Reddy | `password` | 2nd |
| `patient_report_3.pdf` | Emily Thompson | `!@#$%^&` | Within built-in list |

All three files were fully decrypted using nothing more than a hash extractor and a 100-word built-in dictionary — no custom wordlist, no character-by-character brute-forcing. This satisfies **Milestone 2**.

---

### Milestone 3 — Critical Data Exposure (Staff Salaries & Shareholders)

Following the M3 brief to look beyond the obvious content, the `/staff/` and `/old/` paths flagged earlier by `robots.txt` were checked directly.

**`/staff/` — directory listing exposed.** Instead of a proper 403/404, the server returned a full directory listing, exposing `staff/login.php` by name:

<p align="center">
  <img src="./evidence/images/13-recon-staff-directory-listing.png" width="680" alt="Directory listing exposed on /staff/">
</p>

**`/old/` — a far more serious exposure.** The same misconfiguration on `/old/` revealed a publicly downloadable historical database backup, `mediroza_db_backup_2019.sql`:

<p align="center">
  <img src="./evidence/images/14-recon-old-directory-listing.png" width="680" alt="Directory listing exposed on /old/ revealing a SQL backup file">
</p>

The backup's own header comment flagged exactly what it contained — confidential staff and shareholder records:

<p align="center">
  <img src="./evidence/images/15-sqlbackup-staff-table-schema.png" width="620" alt="SQL backup header and staff table schema">
</p>

The dump included full staff records — names, job titles, departments, contact details, national ID numbers, and **monthly salaries** for all 30 hospital employees:

<p align="center">
  <img src="./evidence/images/16-sqlbackup-staff-table-data.png" width="620" alt="Staff table data dump including salary figures">
</p>

...and a separate `shareholders` table listing ownership stakes in the hospital:

<p align="center">
  <img src="./evidence/images/17-sqlbackup-shareholders-table.png" width="620" alt="Shareholders table showing ownership percentages">
</p>

This satisfies **Milestone 3**: both required data points — staff salaries and shareholder details — were fully recovered, sourced from an unauthenticated, publicly accessible backup file rather than any further exploitation of the login form.

> ⚠️ **Handling note:** The dump contains real-format PII (national ID numbers, salaries, contact details). This should be redacted or excluded from any public-facing copy of this repository; it is retained here only as evidence for the training deliverable.

---

### Dead Ends Worth Documenting

Not every lead paid off — and a thorough test log records the negative results too, since they show the surface was actually checked rather than assumed safe.

`/patient/reports/` (a plausible sibling of the working `/patient/portal.php` path) was probed directly, but correctly returned a **403 Forbidden** rather than any listing or data — this path is properly locked down:

<p align="center">
  <img src="./evidence/images/19-recon-patient-reports-403.png" width="620" alt="403 Forbidden response for /patient/reports/">
</p>

An attempt was also made to pull `/patient/error_log` in case a stack trace or misconfigured log had leaked query details, credentials, or file paths. The request itself returned `HTTP/2 403`, and a `grep` for common sensitive keywords (`sql`, `bash`, `shell`, `password`, `error`, `warning`) against the saved response confirmed it was just the standard Forbidden page body — no log content was actually exposed:

<p align="center">
  <img src="./evidence/images/20-recon-error-log-attempt.png" width="620" alt="Attempt to fetch and grep /patient/error_log, returning only the 403 page">
</p>

---

## 📊 Risk Rating Justification

| Finding | Ease of Exploitation | Auth Required | Data Sensitivity | Rating |
|---|---|---|---|---|
| SQL Injection Auth Bypass | Trivial — one payload, browser only | None | Very high (PHI) | 🔴 **Critical** |
| Exposed DB Backup on `/old/` | Trivial — single request | None | Very high (PII + financial) | 🔴 **Critical** |
| Weak PDF Passwords | Trivial — free tool, seconds per file | None (post-access) | Very high (PHI) | 🟠 **High** |
| Directory Listing (`/staff/`, `/old/`) | Trivial — single request | None | High — reveals attack surface & data | 🟠 **High** |
| Username Enumeration | Trivial — observe response text | None | Low direct, enables further attacks | 🟡 **Medium** |

---

## 🛠️ Recommendations (Preview)

- Parameterize all database queries; never concatenate user input into SQL statements.
- Return generic authentication failure messages regardless of whether the username exists.
- Disable directory listing (autoindex) on all web-accessible paths.
- Remove `/old/` from the production webroot entirely — backups do not belong in a public directory.
- Enforce strong, randomly generated passwords (not dictionary words) for any password-protected patient documents, or move to authenticated in-portal viewing instead of downloadable encrypted files.
- Treat `robots.txt` as a public document — never list sensitive paths in it, since it actively invites the reconnaissance it's meant to discourage.

Full remediation detail will be delivered in the Milestone 4 report.

---

## 📁 Repository Structure

```
mediroza-pentest-report/
├── README.md                 ← you are here
└── evidence/
    └── images/                ← 21 source screenshots (00–20), referenced inline above
```

---

**Status:** Milestones 1–3 complete and documented above. Milestone 4 (full formal pentest report) is the final deliverable for this engagement.
