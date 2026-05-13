# Recon - GOAD

**Date :** 2026-04-30
**Statut :** ✅ Done

## Objectif
Découverte du périmètre GOAD : hosts, OS, services, domaines, premiers vecteurs.

---

## 1. Host Discovery

```bash
nmap -sn 192.168.56.0/24
```

```
192.168.56.1   → up (Gateway Linux)
192.168.56.10  → up
192.168.56.11  → up
192.168.56.22  → up
192.168.56.100 → up (tous ports filtrés)
```

---

## 2. Port Scan + OS + Services

```bash
nmap -T5 -sS -sV -O 192.168.56.0/24
```

```
192.168.56.1 — Linux Ubuntu
  22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15
  111/tcp  open  rpcbind 2-4 (RPC #100000)
  2049/tcp open  nfs     3-4 (RPC #100003)

192.168.56.10 — KINGSLANDING — sevenkingdoms.local — Windows Server 2019
  53/tcp   open  domain        Simple DNS Plus
  80/tcp   open  http          Microsoft IIS httpd 10.0
  88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
  135/tcp  open  msrpc         Microsoft Windows RPC
  139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
  389/tcp  open  ldap          Microsoft Windows AD LDAP (Domain: sevenkingdoms.local)
  445/tcp  open  microsoft-ds  signing:TRUE
  464/tcp  open  kpasswd5
  593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
  636/tcp  open  ssl/ldap
  3268/tcp open  ldap          Global Catalog
  3269/tcp open  ssl/ldap      Global Catalog
  3389/tcp open  ms-wbt-server Microsoft Terminal Services
  5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
  5986/tcp open  ssl/wsmans

192.168.56.11 — WINTERFELL — north.sevenkingdoms.local — Windows Server 2019
  53/tcp   open  domain        Simple DNS Plus
  88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
  135/tcp  open  msrpc         Microsoft Windows RPC
  139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
  389/tcp  open  ldap          Microsoft Windows AD LDAP (Domain: sevenkingdoms.local)
  445/tcp  open  microsoft-ds  signing:TRUE
  464/tcp  open  kpasswd5
  593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
  636/tcp  open  ssl/ldap
  3268/tcp open  ldap          Global Catalog
  3269/tcp open  ssl/ldap      Global Catalog
  3389/tcp open  ms-wbt-server Microsoft Terminal Services
  5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
  5986/tcp open  ssl/wsmans

192.168.56.22 — CASTELBLACK — north.sevenkingdoms.local — Windows Server 2019
  80/tcp   open  http          Microsoft IIS httpd 10.0
  135/tcp  open  msrpc         Microsoft Windows RPC
  139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
  445/tcp  open  microsoft-ds  signing:FALSE ← VULNÉRABLE NTLM RELAY
  1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000
  3389/tcp open  ms-wbt-server Microsoft Terminal Services
  5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
  5986/tcp open  ssl/wsmans

192.168.56.100 — Inconnu — tous ports filtrés — OS inconnu
```

---

## 3. SMB Enum null session

```bash
netexec smb 192.168.56.10 192.168.56.11 192.168.56.22
```

```
KINGSLANDING → Windows 10/Server 2019 Build 17763 x64 (name:KINGSLANDING) (domain:sevenkingdoms.local) (signing:True) (SMBv1:False)
WINTERFELL   → Windows 10/Server 2019 Build 17763 x64 (name:WINTERFELL) (domain:north.sevenkingdoms.local) (signing:True) (SMBv1:False)
CASTELBLACK  → Windows 10/Server 2019 Build 17763 x64 (name:CASTELBLACK) (domain:north.sevenkingdoms.local) (signing:False) (SMBv1:False) ← VULN RELAY
```

```bash
netexec smb 192.168.56.10 192.168.56.11 192.168.56.22 -u guest -p ""
```

```
north.sevenkingdoms.local\guest → STATUS_ACCOUNT_DISABLED
north.sevenkingdoms.local\guest → STATUS_ACCOUNT_DISABLED
sevenkingdoms.local\guest       → STATUS_ACCOUNT_DISABLED
```

---

## 4. LDAP Null Session

```bash
netexec ldap 192.168.56.10 192.168.56.11 -u '' -p ''
```

```
WINTERFELL   → [+] north.sevenkingdoms.local\: (bind accepté)
               [-] LdapErr: DSID-0C090A5C — requêtes bloquées sans auth complète
KINGSLANDING → [+] sevenkingdoms.local\: (bind accepté)
               [-] LdapErr: DSID-0C090A5C — requêtes bloquées sans auth complète
```

---

## 5. RPC Null Session

```bash
rpcclient -U "" -N 192.168.56.10 -c "enumdomusers"
```

```
result was NT_STATUS_ACCESS_DENIED
→ KINGSLANDING refuse les sessions anonymes (mieux durci)
```

```bash
rpcclient -U "" -N 192.168.56.11 -c "enumdomusers"
```

```
user:[Guest] rid:[0x1f5]
user:[arya.stark] rid:[0x456]
user:[sansa.stark] rid:[0x45a]
user:[brandon.stark] rid:[0x45b]
user:[rickon.stark] rid:[0x45c]
user:[hodor] rid:[0x45d]
user:[jon.snow] rid:[0x45e]
user:[samwell.tarly] rid:[0x45f]
user:[jeor.mormont] rid:[0x460]
user:[sql_svc] rid:[0x461]

→ WINTERFELL accepte les sessions anonymes RPC (mauvaise config)
→ 10 users énumérés dont sql_svc (compte service MSSQL → Kerberoast possible)
```

---

## 6. DNS Zone Transfer

```bash
dig axfr @192.168.56.10 sevenkingdoms.local
dig axfr @192.168.56.11 north.sevenkingdoms.local
```

```
Transfer failed sur les deux DCs → BLOQUÉ
```

---

## 7. HTTP Recon

```bash
whatweb http://192.168.56.10
whatweb http://192.168.56.22
```

```
192.168.56.10 → [200 OK] IIS/10.0, ASP.NET, Title[IIS Windows Server] — page défaut
192.168.56.22 → [200 OK] IIS/10.0, ASP.NET, sans titre — app custom possible
```

---

## Findings Recon

| Finding | Détail | Impact |
|---------|--------|--------|
| CASTELBLACK SMB signing FALSE | NTLM relay possible | 🔴 Critique |
| RPC null session WINTERFELL | 10 users énumérés | 🟠 Élevé |
| sql_svc exposé | Compte service MSSQL → Kerberoast | 🟠 Élevé |
| MSSQL 2019 port 1433 ouvert | Default creds possible | 🟠 Élevé |
| IIS sur .10 et .22 | Web app à explorer | 🟡 Moyen |
| Guest désactivé partout | Pas d'accès anonyme SMB | ℹ️ Info |
| DNS Zone Transfer bloqué | Pas de cartographie DNS | ℹ️ Info |

---

Tags : #goad #recon #done #pentest
