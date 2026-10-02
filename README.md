# Mediroza General Hospital, Black Box Penetration Test

**Client:** Mediroza General Hospital
**Target:** https://medirozahospital.com
**Engagement Type:** Full Black Box Penetration Test
**Conducted By:** CHUKWU PRAISEGOD
**Engagement Duration:** 5 days
**Authorization:** Written authorization granted by the client for the stated scope and timeline

> This engagement was conducted in a controlled environment for educational purposes only, as part of my Networkwalks Cybersecurity Internship. The target was authorized for security testing by Networkwalks. The techniques documented here must never be applied to any system without explicit written permission from the system owner.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Scope and Rules of Engagement](#scope-and-rules-of-engagement)
- [Tools Used](#tools-used)
- [Methodology](#methodology)
- [Milestone 1: Initial Access](#milestone-1-initial-access)
- [Milestone 2: Data Extraction](#milestone-2-data-extraction)
- [Milestone 3: Critical Data Exposure](#milestone-3-critical-data-exposure)
- [Risk Summary](#risk-summary)
- [Recommendations](#recommendations)
- [Methodology Notes and Limitations](#methodology-notes-and-limitations)
- [Full Report](#full-report)
- [Disclaimer](#disclaimer)

---

## Project Overview

This repository documents a black box penetration test performed against the public facing web infrastructure of Mediroza General Hospital, as part of a structured training engagement with defined milestones. The objective was to demonstrate real world impact by actively identifying and exploiting vulnerabilities rather than relying on automated scanning alone, then tracing that access through to the exposure of confidential patient and business data.

The engagement was broken into four milestones:

| Milestone | Objective |
|---|---|
| M1, Initial Access | Attack the website and retrieve 3 confidential patient PDF lab reports |
| M2, Data Extraction | Crack the encryption on all 3 retrieved files |
| M3, Critical Data Exposure | Find staff salaries and shareholder details of the hospital |
| M4, Reporting | Write a professional penetration testing report for the client |

The result was a complete compromise chain: an unauthenticated SQL Injection vulnerability in the patient login form led to the retrieval of confidential lab reports, which were decrypted using weak password recovery techniques, and whose metadata in turn disclosed a legacy backup directory left exposed on the production server, containing staff salary and shareholder data.

---

## Scope and Rules of Engagement

- **Target:** https://medirozahospital.com and its associated web infrastructure only
- **Testing Type:** Full black box, no prior credentials or internal knowledge provided
- **Allowed:** Active exploitation to demonstrate real impact, within the target domain
- **Not Allowed:** Social engineering, denial of service testing, testing outside the agreed scope
- **Authorization:** Written authorization was provided by the client prior to any testing activity

---

## Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Network and service enumeration |
| Gobuster | Subdomain and directory and file discovery |
| dig, dnsrecon | DNS enumeration and zone transfer testing |
| curl, swaks, netcat | Manual service interaction and protocol level testing |
| Browser Dev Tools / manual testing | Authentication behaviour analysis and SQL Injection testing |
| John the Ripper (pdf2john.pl) | Extracting crackable hash representations of PDF encryption |
| Hashcat | Offline password recovery against PDF encryption hashes |
| ExifTool | PDF metadata analysis |
| rockyou.txt | Primary password wordlist |

---

## Methodology

1. **Reconnaissance:** Network and service enumeration, DNS enumeration, directory and content discovery against the live application.
2. **Entry Point Identification:** Located authentication protected areas of the application and analysed their behaviour under invalid, guessed, and default credentials.
3. **Exploitation:** Exploited a SQL Injection authentication bypass to access a restricted patient area and retrieve confidential files.
4. **Data Extraction:** Extracted and cracked PDF encryption passwords to recover the contents of the retrieved files.
5. **Follow Up Analysis:** Analysed recovered file metadata, which led directly to a further critical exposure on the server.
6. **Reporting:** Documented all actions, evidence and recommendations in a professional report.

Reconnaissance activity that did not contribute to the exploitation path, such as SMTP open relay and VRFY enumeration testing, returned negative results and is noted briefly in the full report as evidence of correctly hardened mail services, but is not treated as a standalone finding here.

---

## Milestone 1: Initial Access

**Objective:** Attack the website and retrieve 3 confidential patient PDF lab reports.

### Entry Points Identified

Two authentication protected areas were identified, both served by a bespoke CMS identified via page metadata as `Mediroza CMS 1.4.2`:

- Patient Portal: `https://medirozahospital.com/patient/login.php`
- Staff Login: `https://medirozahospital.com/staff/login.php`

### Authentication Behaviour Analysis

Testing invalid, default and guessed credentials against both forms revealed a username enumeration weakness specific to the Patient Portal:

| Input | Patient Portal Response | Staff Login Response |
|---|---|---|
| Empty fields | `username not found` | `invalid username or password` |
| `test` / `test` | `username not found` | `invalid username or password` |
| `patient` / `patient` | `username not found` | `invalid username or password` |
| `admin` / `admin` | `incorrect password` | `invalid username or password` |

The differing response for `admin` confirmed a valid account by that name exists on the Patient Portal, and indicated the backend performs a database lookup on the username before validating the password, a pattern commonly associated with unsanitised SQL query construction. The Staff Login form, by contrast, returned a single generic error for every case tested, with no enumeration possible.

### Exploitation: SQL Injection, Authentication Bypass

Based on the above, the following payload was submitted in the Patient Portal username field:

```
Username: admin'--
Password: (any value)
```

This bypassed authentication entirely and granted access to the patient dashboard with no valid password, consistent with a backend query structured similarly to:

```sql
SELECT * FROM users WHERE username = 'admin'--' AND password = '...'
```

The injected `--` comments out the remainder of the query, including the password check.

### Result

Access was gained to a restricted patient area containing three confidential PDF lab reports, which were downloaded:

- `patient_report_1.pdf`
- `patient_report_2.pdf`
- `patient_report_3.pdf`

**Risk Rating:** Critical (SQL Injection / Authentication Bypass), Medium (Username Enumeration)

### Evidence 

**Proof of access** 
![patient dashboard](/screenshots/proof-of-access.png)

**The 3 retrieved PDF files** 
![retrieved files](/screenshots/3-retrieved-pdf-files.png)


---

## Milestone 2: Data Extraction

**Objective:** Crack the encryption on all 3 retrieved files.

All three PDF files were protected with standard PDF encryption, confirmed as RC4 128 bit with a required user password (PDF encryption revision V2, R3).

### Hash Extraction

```bash
perl /usr/share/john/pdf2john.pl patient_report_1.pdf > patient_report_1.hash
perl /usr/share/john/pdf2john.pl patient_report_2.pdf > patient_report_2.hash
perl /usr/share/john/pdf2john.pl patient_report_3.pdf > patient_report_3.hash
```

The leading filename prefix written by `pdf2john.pl` (for example `patient_report_1.pdf:`) was manually removed from each hash file before it could be accepted by Hashcat.

### Password Cracking

```bash
hashcat -m 10500 patient_report_1.hash /usr/share/wordlists/rockyou.txt
hashcat -m 10500 patient_report_2.hash /usr/share/wordlists/rockyou.txt
hashcat -m 10500 patient_report_3.hash /usr/share/wordlists/rockyou.txt
```

Mode `10500` corresponds to 128 bit RC4 encrypted PDF files. All three passwords were successfully recovered using the `rockyou.txt` wordlist.

### Result

All three confidential lab reports were successfully decrypted and their contents recovered, confirming that the password protection applied to this patient health information could be defeated within minutes using a freely available wordlist.

**Risk Rating:** High (Weak, Crackable PDF Encryption Passwords)

### Evidence

**Hashcat output showing cracked status and recoveredd plaintext password for each of the PDF files**
![cracked pdf passwords](/screenshots/cracked-pdf-passwords.png)

**Patient report 1 decrypted**
![decrypted pdf 1](/screenshots/patient_report_1.png)

**Patient report 2 decrypted**
![decrypted pdf 2](/screenshots/patient_report_2.png)

**Patient report 3 decrypted**
![decrypted pdf 3](/screenshots/patient_report_3.png)

---

## Milestone 3: Critical Data Exposure

**Objective:** Find the salaries of all hospital employees and the shareholder details of the hospital.

### Metadata Analysis

With the decryption passwords in hand, the metadata of all three PDF files was reviewed:

```bash
exiftool -password <recovered_password> patient_report_1.pdf
```

This revealed an internal operational note left in the document's Comment metadata field:

```
Comment: DB backup moved to /old before site migration, do not delete
```

This directly disclosed the existence and relative location of a legacy backup directory on the production web server.

### Follow Up Access

The `/old/` directory referenced in the metadata had already been observed during initial reconnaissance in Milestone 1, where it returned a redirect response during directory enumeration but had not yet been examined manually. Revisiting it directly:

```
https://medirozahospital.com/old/
```

confirmed the directory was fully accessible with no authentication whatsoever.

### Result

The following highly sensitive business documents were retrieved without restriction:

- Salary details for all hospital employees
- Shareholder details for Mediroza General Hospital

This represents a complete failure to properly decommission data following a prior site migration, directly exposing personal payroll data and confidential ownership information to any unauthenticated party capable of locating the path.

**Risk Rating:** Critical (Sensitive Data Exposure via Legacy Backup Directory), Medium (Information Disclosure via PDF Metadata)

### Evidence

**Metadata showing critical data exposure**
![Report 3 metadata](/screenshots/patient_report_3_metadata.png)

**Critical exposure on the server page1**
![Critical server exposure 1](/screenshots/critical-exposure-1.png)

**Critical exposure on the server page2**
![Critical server exposure 2](/screenshots/critical-exposure-2.png)

---

## Risk Summary

| Finding | Milestone | Risk Rating |
|---|---|---|
| SQL Injection leading to Authentication Bypass on Patient Portal | M1 | Critical |
| Username Enumeration on Patient Portal Login | M1 | Medium |
| Weak, Crackable PDF Encryption Passwords | M2 | High |
| Sensitive Data Exposure via Legacy Backup Directory (`/old/`) | M3 | Critical |
| Information Disclosure via PDF Metadata | M3 | Medium |

---

## Recommendations

**SQL Injection and Authentication Bypass**
- Rebuild all database queries using parameterised queries or prepared statements
- Apply strict server side input validation on all authentication fields
- Review the CMS authentication module for other instances of the same pattern
- Deploy a Web Application Firewall as a compensating control during remediation

**Username Enumeration**
- Return a single generic error message regardless of whether the username exists
- Implement rate limiting and account lockout on the Patient Portal login

**Weak PDF Encryption Passwords**
- Enforce a strong, unique password policy for any distributed patient documents
- Upgrade encryption from RC4 128 bit to AES 256 bit
- Where feasible, deliver reports through an authenticated portal instead of password protected attachments

**PDF Metadata and Document Hygiene**
- Strip internal comments and notes from documents before distribution
- Introduce an automated sanitisation step in the report generation process

**Legacy Backup Directory Exposure**
- Immediately remove the `/old/` directory and any other legacy content from the production web server
- Establish a formal data migration and decommissioning checklist
- Store all backups outside the public web root, with encryption and access control at rest
- Audit the web root for any other forgotten or legacy directories

**General**
- Schedule regular penetration testing, particularly after future migrations or major changes
- Provide secure coding training to development staff
- Maintain a reviewed inventory of all directories and files on the production server

Full justification for each risk rating and a complete set of recommendations is provided in the attached report.

---

## Methodology Notes and Limitations

Hash extraction and cracking for the three retrieved PDF files was initially attempted using Networkwalks' own online hash calculator and password cracker utilities. This recovered the passwords for `patient_report_1.pdf` and `patient_report_2.pdf`, but could not recover the password for `patient_report_3.pdf` due to the limited wordlist available to that online tool.

The process was repeated locally using `pdf2john.pl` and Hashcat with `rockyou.txt`, which successfully recovered all three passwords. For engagements involving genuinely confidential or regulated data, hash extraction and cracking should be performed entirely with local, offline tooling from the outset, rather than submitting sensitive files or derived hashes to third party online services.

---

## Full Report

The complete professional penetration testing report, including detailed proof of exploitation, a full risk rating table with justification, and complete remediation guidance, is included in this repository:

**[Mediroza_Pentest_Report.pdf](./Mediroza_Pentest_Report.pdf)**

---

## Disclaimer

This project was conducted in a controlled environment for educational purposes only, as part of a Networkwalks training engagement. All testing was performed against infrastructure explicitly authorized for security testing, within the agreed scope and rules of engagement. The techniques, payloads and findings documented in this repository must never be applied to any system without explicit written permission from the system owner. Any data referenced as recovered during this engagement (patient reports, salary records, shareholder details) relates to a controlled training target and is documented here solely to demonstrate the technical findings of the assessment.
