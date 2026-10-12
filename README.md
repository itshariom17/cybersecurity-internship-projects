# Cybersecurity Internship Projects

**Author: Hari Om**  
**Date: 12 October 2026**  
**Selected projects: 1. VAPT and 7. Password Cracking and Hashing Algorithms**

Two reproducible educational Python projects using only the standard library. They demonstrate concepts relevant to penetration-testing study; they are not a full OSCP practice environment or a claim of OSCP certification. Development was assisted by AI. Review the code and reproduce the experiments before presenting them.

## Project 1: Local Web Application VAPT

### Brief explanation

A small notes API deliberately contains two security flaws. The assessment demonstrates them, applies focused fixes, and repeats the same requests to verify remediation. All requests target a temporary server on 127.0.0.1 created by the assessment itself.

### Objective and scope

Identify and exploit authentication and authorization weaknesses in a simulated application, document their impact, and verify fixes. Two synthetic users, Alice and Bob, each own a note. No real accounts, remote targets, customer records, or external systems are used.

### Components and workflow

- `vapt/lab.py`: Python HTTP server, in-memory SQLite database, bearer sessions, login and notes endpoints. Defaults to fixed mode and binds only to loopback.
- `vapt/assessment.py`: starts vulnerable and fixed instances, sends nine HTTP requests to each, records sanitized responses, and shuts down both servers.
- `vapt/results.json`: actual request/response evidence from the completed assessment. Session tokens are redacted.
- `tests/test_projects.py`: regression checks for exploits, fixes, authentication, and invalid object identifiers.

Workflow: define scope → enumerate the service → check normal behavior → demonstrate the flaws → apply fixes → retest → report.

### Findings and observed results

| Finding | Lab severity | Vulnerable result | Fixed result | Remediation |
|---|---|---|---|---|
| SQL injection in login (CWE-89) | High | Crafted username authenticated Alice despite an incorrect password; HTTP 200 | Same request rejected; HTTP 401 | Replace string-built SQL with a parameterized query |
| Insecure direct object reference (CWE-639) | Medium | Alice retrieved Bob's synthetic note; HTTP 200 | Same request denied; HTTP 403 | Check note ownership against the authenticated session |

Severity is a qualitative assessment of this lab, not a calculated CVSS score or a production risk assessment.

**SQL injection reproduction:** send POST `/login` with username `alice' -- ` and password `wrong`. In vulnerable mode, the comment marker removes the password condition from the SQL statement. In fixed mode, the entire username is treated as data and no account matches.

**IDOR reproduction:** log in as Alice using the published lab credentials, then request `/notes?id=2` with Alice's bearer token. Vulnerable mode checks authentication but omits ownership validation. Fixed mode returns 403.

Control requests confirm that valid login and access to one's own note still work; unauthenticated access and invalid passwords return 401; nonexistent notes return 404; malformed note identifiers return 400.

### Run it

Use Python 3.10 or newer from the repository root; validation here used Python 3.12. No package installation is required.

```console
python -m vapt.assessment
```

For manual inspection in a separate terminal:

```console
python -m vapt.lab --mode vulnerable --port 8000
```

Open `http://127.0.0.1:8000/` to inspect the route listing. Stop with Ctrl+C. To inspect the remediated version, rerun with `--mode fixed`. The assessment runner manages its own servers and does not need this manual server.

Lab credentials: `alice` / `Alice-lab-only!` and `bob` / `Bob-lab-only!`. These are intentional public fixtures, never real credentials.

### Limits and learning outcomes

This is a focused web VAPT demonstration, not a comprehensive assessment. It teaches HTTP enumeration, SQL injection, access-control testing, evidence capture, remediation, and retesting. It does not cover host privilege escalation, Active Directory, pivoting, or an entire OSCP exam workflow.

The fixed mode only repairs the two demonstrated flaws. It remains a training API: plaintext synthetic passwords in memory, no TLS, no session expiry/logout, no rate limiting, and a basic HTTP server. Do not deploy either mode to a public host. Nmap, Burp Suite, Nessus, Metasploit, and Kali were not used in the recorded execution; the implementation uses Python, SQLite, and actual local HTTP requests.

## Project 7: Password Cracking and Hashing Laboratory

### Brief explanation

This offline experiment creates its own test hashes and attempts to recover deliberately weak passwords. It compares SHA-256, salted SHA-256, and PBKDF2-HMAC-SHA256, then measures verification time and demonstrates why salting and computational cost serve different purposes.

### Components and method

- `password_lab/passwords.py`: hash creation, constant-time digest comparison, bounded dictionary search, bounded brute-force candidate generation, salt comparison, and timing measurements.
- `password_lab/results.json`: actual generated hashes, public synthetic passwords, attempt counts, and local timings.
- `tests/test_projects.py`: known SHA-256 vector, correct/incorrect verification, salt uniqueness, success/failure cases, search limits, brute force, and Unicode.

The dictionary contains five synthetic candidate words. The target `lab-only-42` appears fourth. A separate three-digit target `042` is searched in numeric order from `000` through `999`. There are no network logins or leaked password datasets.

PBKDF2 uses a randomly generated 16-byte salt and 600,000 iterations. Five verification timings per algorithm are collected and the median is reported. The dictionary search has a configurable attempt cap (10,000 by default); brute-force generation rejects spaces above one million candidates.

### Recorded results

| Experiment | Observed outcome |
|---|---|
| Dictionary search: SHA-256 | Recovered `lab-only-42` in 4 attempts |
| Dictionary search: salted SHA-256 | Recovered the same password in 4 attempts |
| Dictionary search: PBKDF2 | Recovered the same password in 4 attempts, taking approximately 0.60 seconds |
| Three-digit brute-force search | Recovered `042` in 43 attempts |
| Password absent from dictionary | No match after 5 attempts |
| Same password with two random salts | Different salts and digests |
| Median SHA-256 verification | Approximately 0.0000013 seconds |
| Median salted SHA-256 verification | Approximately 0.0000014 seconds |
| Median PBKDF2 verification | Approximately 0.151 seconds |

These are single-process CPU measurements on one run, not GPU cracking rates. Microsecond timings are noisy; do not interpret the small difference between the two SHA variants as a stable performance ranking. Times and randomly salted digests change on reruns.

### What the results mean

Salting prevents identical passwords from producing identical stored hashes across records and reduces the value of precomputed tables. It does not stop guessing a weak password when the salt is known. PBKDF2 raises the cost of each guess, but a password in a short dictionary remains recoverable. Fast SHA-256, even with a salt, is not a suitable production password-storage scheme.

OWASP recommends modern password hashing such as Argon2id; this standard-library project uses PBKDF2 to demonstrate an adjustable work factor without external dependencies. The 600,000 iteration setting follows OWASP's PBKDF2-HMAC-SHA256 guidance. Choosing production settings also requires performance testing and operational design.

### Run it

```console
python -m password_lab.passwords
```

This regenerates `password_lab/results.json`. The command runs only the included synthetic demonstration; there is no remote target or arbitrary hash-file CLI. Hashcat and John the Ripper were not used in the recorded results.

## Validation

```console
python -m unittest discover -s tests -v
```

**Recorded verification: 12 tests passed** on 12 October 2026 (8 password tests and 4 VAPT tests). The VAPT assessment additionally recorded 18 HTTP exchanges, nine in each mode. Network permission is needed for loopback HTTP tests in restricted environments.

## Short presentation script

“My name is Hari Om. My first project demonstrates a local web application vulnerability assessment. I reproduced SQL injection and unauthorized access to another user's note, then verified focused fixes with repeatable tests. My second project compares password hashing methods through bounded offline dictionary and brute-force experiments on synthetic data. The results show that salts separate hash records and PBKDF2 makes each guess more expensive, while weak passwords still remain guessable.”

## References

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [OffSec PEN-200 syllabus](https://manage.offsec.com/app/uploads/2026/03/PEN-200_Syllabus.pdf)

## Submission status

Local implementation and validation are complete. [LinkedIn project announcement](https://www.linkedin.com/feed/update/urn:li:activity:7515396164029059072/). The offer letter is a separate private submission attachment and is not included in this project folder.
