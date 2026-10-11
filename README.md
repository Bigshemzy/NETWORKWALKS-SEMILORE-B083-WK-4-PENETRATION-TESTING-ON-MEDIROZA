# NETWORKWALKS-SEMILORE-B083-WK-4-PENETRATION-TESTING-ON-MEDIROZA
This is a penetration test on Mediroza website as part of my internship project on networkwalks

# Penetration Testing Project — Mediroza General Hospital

## Overview
A black box penetration test conducted against `medirozahospital.com` as part of the structured training engagement with Networkwalks. The objective was to identify vulnerabilities in the hospital's web infrastructure, exploit them to demonstrate real-world impact, and produce a professional penetration testing report.

**Engagement Type:** Black-box Penetration Test  
**Target:** https://medirozahospital.com  
**Authorization:** Written permission granted by client for the duration of the engagement  
**Scope:** Testing limited to the target domain only — no social engineering, no denial-of-service, no testing outside agreed scope  

---

## Milestone 1 — Initial Access

**Goal:** Attack the website and retrieve 3 confidential patient PDF lab reports.

### Reconnaissance
Started with a check of `robots.txt`, which disclosed several directories the site owner did not want indexed:

```bash
curl https://medirozahospital.com/robots.txt
```
```
Disallow: /patient/
Disallow: /staff/
Disallow: /old/
```

The `/patient/` directory stood out as the likely entry point for this milestone.

### Discovering the Login Page
Navigating to `/patient/` revealed a login portal at `/patient/login.php`.

### Username Enumeration
Testing the login form with a fake username versus a guessed real one (`admin`) returned **different error messages** — "Username not found" vs. "Incorrect password." This inconsistency confirms the account `admin` exists and is a textbook username enumeration vulnerability.

### SQL Injection
Submitting a single quote (`'`) in the username field triggered a raw MySQL syntax error, confirming the login form was vulnerable to SQL injection — user input was being inserted directly into the query without sanitization.

### Authentication Bypass
Using the classic payload:
```
username: admin' --
password: anything
```
The `'--` sequence breaks out of the SQL query and comments out the rest, including the password check — logging in as `admin` with no valid password required.

**Result:** Successful authentication bypass. The portal loaded 3 patient lab report PDFs, which were downloaded.

---

## Milestone 2 — Crack the Encryption

**Goal:** Crack the encryption on all 3 retrieved PDF files.

Each PDF was encrypted with a different password strength, reinforcing that a single cracking approach doesn't always work.

| File | Method | Result |
|------|--------|--------|
| `patient_report_1.pdf` | Hash extraction + common-password wordlist | `123456` |
| `patient_report_2.pdf` | Hash extraction + common-password wordlist | `password` |
| `patient_report_3.pdf` | Common wordlist failed → larger wordlist (JtR list) | `!@#$%^&` |

The third file required escalating to a larger, more comprehensive wordlist after the initial attempt exhausted without a match — a useful reminder that real-world password audits require multiple wordlists of varying size and specificity.

Once cracked, the file was decrypted for further analysis:
```bash
qpdf --password='!@#$%^&' --decrypt patient_report_3.pdf report3_open.pdf
```

---

## Milestone 3 — Deep Reconnaissance

**Goal:** Find the critical data exposure on the client server.

### Metadata Analysis
Running `exiftool` against the unlocked PDF revealed hidden metadata fields:
```bash
exiftool report3_open.pdf
```
```
Author   : j.malik
Comments : DB backup moved to /old before site migration, do not delete
```

A staff member had left an internal note embedded in the file's metadata, pointing directly to a forgotten backup location — `/old/`, the same directory flagged earlier in `robots.txt`. This is a strong example of how two independent pieces of recon (metadata + robots.txt) converged on the same finding.

### Directory Listing Exposure
Navigating to `https://medirozahospital.com/old/` revealed that **directory listing was enabled** — exposing the full contents of the folder, including:
```
mediroza_db_backup_2019.sql
```

```bash
wget https://medirozahospital.com/old/mediroza_db_backup_2019.sql
```

### Analyzing the Database Backup
The `.sql` backup contained full `INSERT INTO` statements for two tables:

- **`staff`** — 30 employee records including full names, job titles, departments, email addresses, phone numbers, **national ID numbers**, and monthly salaries
- **`shareholders`** — ownership records showing shareholder names, share percentages, shares held, and share class

### Connecting the Chain
Cross-referencing the metadata `Author` field (`j.malik`) against the staff table identified **Jameel Malik, IT Systems Administrator** — the employee who moved the backup and left the note. This traced the full attack chain from a stray metadata comment to a complete confidential data breach, showing how a single careless internal note led directly to exposure of salary, PII, and ownership data.

---

## Milestone 4 — Findings Summary & Risk Rating

| # | Vulnerability | Location | Risk |
|---|----------------|----------|------|
| 1 | Username enumeration on login page | `/patient/login.php` | Medium |
| 2 | SQL injection login bypass | `/patient/login.php` | Critical |
| 3 | Encrypted PDFs accessible after login bypass | `/patient/reports/` | High |
| 4 | Weak PDF passwords crackable with a wordlist | `patient_report_*.pdf` | High |
| 5 | Sensitive metadata left in patient PDF files | `patient_report_3.pdf` | Medium |
| 6 | Forgotten backup folder with directory listing enabled | `/old/` | Critical |
| 7 | Confidential staff salaries, PII, and shareholder data in plaintext | `/old/mediroza_db_backup_2019.sql` | Critical |

## Recommendations

- **Username enumeration:** Return identical error messages regardless of whether the username or password is incorrect.
- **SQL injection:** Use parameterized queries / prepared statements — never concatenate raw user input into SQL.
- **PDF access control:** Store sensitive documents outside the web root or behind proper authentication; enforce strong, unique passwords per file.
- **PDF metadata hygiene:** Strip metadata before distributing documents externally (`exiftool -all= filename.pdf`).
- **Directory listing & backup exposure:** Disable directory listing globally; never store database backups inside a publicly accessible web folder; relocate or securely archive legacy backups.

---

## Key Takeaways

This engagement demonstrated how individually "minor" findings compound into critical exposure:
- A verbose login error message led to confirmed valid usernames.
- Unsanitized input led to a full authentication bypass.
- Weak PDF passwords exposed protected documents in minutes.
- A single overlooked metadata comment exposed an entire unprotected database backup containing PII, salaries, and ownership records.

It reinforced the value of **thorough, methodical reconnaissance** — the same clue (`/old/`) surfaced independently through both `robots.txt` enumeration and file metadata analysis, showing why cross-referencing findings across different recon techniques matters in real engagements.

## Challenges Encountered

A significant obstacle during initial recon was a **JavaScript-based "browser verification" challenge** sitting in front of the main site on ports 80/443. Every non-browser tool used — `curl`, `whatweb`, `sqlmap`, `gobuster` — received the same generic "verifying your browser" interstitial page instead of real site content, since none of these tools execute JavaScript.

This had a direct, cascading effect on recon accuracy:
- `robots.txt` and `sitemap.xml` returned the challenge page rather than their actual contents, masking the `Disallow: /patient/ /staff/ /old/` entries that were the real starting point for the entire engagement.
- `sqlmap` reported "no forms found" because it only ever saw the JS-challenge wrapper, not the real login form.
- `gobuster` results were unreliable, since the challenge page was served on virtually every path, making it difficult to distinguish genuine directories from false positives.

**Lesson learned:** When automated tooling is met with suspiciously uniform responses across every request (same title, same content length, same status code regardless of path), that's a strong signal of bot-protection middleware rather than a real 404 or empty result. The correct response is to verify manually in a real browser first — letting the JS challenge resolve — and then either continue recon manually or export the resulting session cookies for reuse in command-line tools (`curl -b`, `gobuster -c`), rather than assuming the tool output reflects the actual site.

This is a realistic obstacle in modern pentesting, where CDN-level bot protection (Cloudflare, OpenResty-based WAFs, etc.) is increasingly common even on smaller sites, and recognizing it early saves significant wasted effort chasing dead ends elsewhere (in this case, time spent probing the cPanel/WHM ports as an alternative entry point).

## Tools Used
`curl` · Burp-style manual SQLi testing · `exiftool` · `qpdf` · John the Ripper–style wordlist cracking · `wget` · `whatweb` · `theHarvester` · `gobuster` · `nmap`

## Disclaimer
This engagement was conducted in a controlled training environment with explicit written authorization from the target owner, as part of a structured Networkwalks training program. These techniques must never be applied to any system without explicit written permission from the owner.
