# BloodHound — Studi Kasus GOAD-Light (Detail Lengkap)

Panduan penggunaan BloodHound untuk memetakan & menganalisis jalur serangan di GOAD-Light (2 domain: `sevenkingdoms.local` + child `north.sevenkingdoms.local`).

> **Konsep dasar:** BloodHound memodelkan Active Directory sebagai **graf**: *node* (User, Group, Computer, GPO, Domain) dihubungkan *edge* (MemberOf, GenericAll, ForceChangePassword, HasSession, dsb.). Dari graf ini kita mencari **jalur serangan** dari akun yang sudah dikuasai menuju Domain Admin.

---

## 1. Persiapan (Neo4j + BloodHound)

### Perintah
```bash
# Start database Neo4j
sudo neo4j console          # atau: sudo neo4j start
# default creds: neo4j / neo4j (ganti saat login pertama)

# Jalankan BloodHound (GUI)
bloodhound
#   login: bolt://localhost:7687  user: neo4j  pass: <password>

# Alternatif web (BloodHound CE):
# sudo docker run -p 7474:7474 -p 7687:7687 -e NEO4J_AUTH=neo4j/password neo4j
```

### Penjelasan teknis
- **Neo4j** = database graf tempat data AD disimpan. BloodHound hanya antarmuka visual/query.
- `bolt://localhost:7687` = protokol koneksi BloodHound → Neo4j.
- BloodHound CE (Community Edition) berjalan di browser dan bisa dipakai tanpa GUI desktop.

---

## 2. Collection (bloodhound-python)

### Tujuan
Menarik data AD (user, group, ACL, trust, session) dari kedua domain.

### Perintah
```bash
# 2.1) Collect domain NORTH
bloodhound-python --zip -c All -d north.sevenkingdoms.local \
  -u brandon.stark -p iseedeadpeople \
  -dc winterfell.north.sevenkingdoms.local -ns 192.168.56.11

# 2.2) Collect domain ROOT sevenkingdoms.local (via trust, UPN suffix)
bloodhound-python --zip -c All -d sevenkingdoms.local \
  -u brandon.stark@north.sevenkingdoms.local -p iseedeadpeople \
  -dc kingslanding.sevenkingdoms.local -ns 192.168.56.10
```

### Penjelasan teknis per flag
- `--zip` : hasil dikemas jadi `.zip` (siap di-import ke BloodHound).
- `-c All` : jalankan **semua collector** (penggabungan SharpHound: `Sessions`, `LoggedOn`, `Groups`, `ACLs`, `Trusts`, `GPO`, `ObjectProps`). Ini menghasilkan graf paling lengkap.
- `-d` : domain target.
- `-u/-p` : kredensial (harus user domain, di sini `brandon.stark`).
- `-dc` : FQDN DC; `-ns` : nameserver/IP DC.

### Perbedaan 2.1 vs 2.2
- 2.1 memakai format user biasa (`brandon.stark`) — domain asalnya north.
- 2.2 memakai **UPN suffix** (`brandon.stark@north.sevenkingdoms.local`) — memanfaatkan **trust** child→parent untuk mengoleksi domain root `sevenkingdoms.local`.

### Hasil
File `*.zip` (JSON) di direktori kerja, siap di-import.

---

## 3. Import & Mark Owned

### Langkah
1. BloodHound → **Upload Data** → pilih kedua `.zip` → import.
2. Tandai principal yang sudah dikompromi:
   - Klik kanan user → **Mark User as Owned** (ikon tengkorak).
   - Contoh: `brandon.stark`, `jon.snow`, `hodor` (hasil spray).

### Penjelasan teknis
- **"Owned"** adalah penanda manual bahwa akun tersebut sudah berada di tangan attacker (punya password/hash).
- Fungsinya: query **`Shortest Path from Owned Principals`** menghitung jalur realistis *dari posisi kita sekarang* ke target — bukan jalur teoretis dari user acak.

### Poin penting
Selalu perbarui penanda "Owned" setiap kali mendapat akun baru (hasil ASREP, kerberoast, spray, dsb.) agar analisis selalu akurat.

---

## 4. Query Bawaan (Built-in)

| Query | Tujuan |
|---|---|
| `Find all Domain Admins` | daftar target DA |
| `Shortest Paths to Domain Admins` | jalur terpendek ke DA |
| `Shortest Path from Owned Principals` | jalur dari akun yang sudah dikuasai |
| `Shortest Path from Domain Users to Domain Admins` | jalur dari user biasa |
| `Find Kerberoastable Users` | SPN (→ `jon.snow`) |
| `Find AS-REP Roastable Users` | tanpa pre-auth (→ `brandon.stark`) |
| `Find Principles with DCSync Rights` | hak GetChanges/GetChangesAll |
| `Find Users with Foreign Group Membership` | cross-domain |

### Penjelasan teknis
- `Find Kerberoastable` = user dengan atribut `servicePrincipalName` (bisa di-kerberoast).
- `Find AS-REP Roastable` = user dengan flag `DONT_REQUIRE_PREAUTH`.
- `Find Principles with DCSync Rights` = siapa yang bisa replikasi NTDS (target untuk DCSync).

---

## 5. Analisis Jalur GOAD-Light (Custom Cypher)

Cypher = bahasa query graf (seperti SQL untuk graf).

### 5.1 ASREP & Kerberoast (entry point)
```cypher
MATCH (u:User {dontreqpreauth:true}) RETURN u           -- ASREP (brandon.stark)
MATCH (u:User {hasspn:true}) RETURN u                   -- Kerberoast (jon.snow)
```
Penjelasan: `dontreqpreauth` dan `hasspn` adalah **property** node User hasil collector. Mencari property ini = menemukan target roast secara visual.

### 5.2 Jalur dari user ke DA NORTH
```cypher
MATCH p=shortestPath((u:User {name:'BRANDON.STARK@NORTH.SEVENKINGDOMS.LOCAL'})-[:MemberOf|AdminTo|HasSession|GenericAll|WriteDacl|ForceChangePassword|Owns|GpLink*1..]->(g:Group {name:'DOMAIN ADMINS@NORTH.SEVENKINGDOMS.LOCAL'}))
RETURN p
```
Penjelasan: `shortestPath` mencari rute terpendek dari `brandon.stark` ke group DA, melewati **edge-edge tertentu** (`MemberOf`, `AdminTo`, `HasSession`, `GenericAll`, dst.). Hasilnya menunjukkan "lompatan" yang harus dieksploitasi.

### 5.3 GPO Abuse (samwell.tarly → STARKWALLPAPER)
```cypher
MATCH (u:User)-[:GenericAll|WriteDacl|WriteOwner|Owns|GenericWrite]->(gpo:GPO)
RETURN u.name, gpo.name
```
Penjelasan: mencari user yang punya **kontrol tulis** atas GPO. Di GOAD-Light: `samwell.tarly` → GPO `STARKWALLPAPER`.

### 5.4 ACL pada Domain Admins sevenkingdoms (endgame)
```cypher
MATCH p=(n)-[r:GenericAll|WriteOwner|WriteDacl|GenericWrite|AddMember]->(g:Group {name:'DOMAIN ADMINS@SEVENKINGDOMS.LOCAL'})
RETURN p
```
Penjelasan: mencari **semua node** yang punya edge berbahaya menuju grup Domain Admins.
> Hasil: `lord.varys` (GenericAll), `petyr.baelish` (WriteProperty), `maester.pycelle` (WriteOwner), `stannis.baratheon` (self-membership).

### 5.5 RBCD ke kingslanding (stannis)
```cypher
MATCH (u:User)-[r:GenericAll|WriteDacl|WriteOwner|GenericWrite]->(c:Computer {name:'KINGSLANDING.SEVENKINGDOMS.LOCAL'})
RETURN u.name, r.isacl, type(r)
```
Penjelasan: `GenericAll` pada **objek Computer** (khususnya DC) memungkinkan **Resource-Based Constrained Delegation** → takeover DC.

### 5.6 Rantai ACL root (full path)
```cypher
MATCH p=(a:User {name:'TYWIN.LANNISTER@SEVENKINGDOMS.LOCAL'})-
  [:ForceChangePassword|GenericWrite|WriteDacl|AddMember|WriteOwner|GenericAll*1..8]->
  (b:Group {name:'DOMAIN ADMINS@SEVENKINGDOMS.LOCAL'})
RETURN p
```
Penjelasan: mencari **jalur penuh** dari `tywin.lannister` ke DA, lewat rantai ACL (maksimal 8 hop).
> tywin → jaime → joffrey → tyron → Small Council → Dragonstone → Kingsguard → stannis → kingslanding → DA.

### 5.7 Cross-domain / trust
```cypher
MATCH p=(d:Domain)-[:TrustedBy]->(d2:Domain) RETURN p
MATCH (u:User)-[:MemberOf*1..]->(g:Group)-[:MemberOf*1..]->(g2:Group)
WHERE g2.name STARTS WITH 'ACROSS' OR g2.name STARTS WITH 'ACROSSTHE'
RETURN u.name, g2.name
```
Penjelasan: (1) tampilkan relasi **trust** antar domain; (2) cari user yang anggota group **cross-domain** (`ACROSSTHENARROWSEA`, `ACROSSTHESEA`).

---

## 6. Walkthrough Analisis (hasil yang diharapkan)

1. **Owned = brandon.stark** (hasil ASREP) → query `Shortest Path from Owned Principals` menuju DA North.
2. **Kerberoastable = jon.snow** → setelah crack, mark owned → jalur `jon.snow` ke DA.
3. **GPO**: `samwell.tarly` punya kontrol atas GPO `STARKWALLPAPER` → RCE di mesin yang menerapkan GPO (winterfell).
4. **Cross-domain**: user north → grup `ACROSSTHENARROWSEA`/`ACROSSTHESEA` → akses ke sevenkingdoms.
5. **Endgame root**: pilih jalur terpendek dari `lord.varys`/`petyr.baelish`/`maester.pycelle`/`stannis` ke **Domain Admins sevenkingdoms** — biasanya GenericAll/WriteOwner langsung.

### Poin penting
Baca edge dengan teliti: `GenericAll` (kontrol penuh) berbeda efek eksploitasinya dengan `ForceChangePassword` (reset password) atau `MemberOf` (keanggotaan langsung).

---

## 7. Eksploitasi Setelah BloodHound

| Temuan BH | Tool eksploitasi |
|---|---|
| ASREP-Roastable | `GetNPUsers.py` + hashcat |
| Kerberoastable | `GetUserSPNs.py` + hashcat |
| GenericAll user | `pywhisker` (shadow creds) / reset password |
| GenericAll/WriteOwner grup DA | `net rpc group addmem` |
| ForceChangePassword | `Set-DomainUserPassword` (PowerView) |
| GPO control | `pyGPOabuse` |
| GenericAll komputer DC | `rbcd.py` / `getST.py` → RBCD |
| DCSync right | `secretsdump.py -just-dc` |

### Penjelasan singkat
- **GenericAll user** → bisa reset password / pasang shadow credentials.
- **GenericAll/WriteOwner pada group DA** → tambahkan diri ke DA.
- **GenericAll komputer DC** → RBCD → impersonate Administrator.
- **DCSync right** → tarik seluruh hash domain.

---

## Catatan Pengajar

- Collection pakai `brandon.stark` sudah cukup; gunakan UPN suffix saat collect domain root (trust).
- Urutkan: import → mark owned → mulai dari `Shortest Path from Owned Principals` (jalur realistis).
- Ajarkan membaca node/edge: warna & jenis edge (`GenericAll`, `ForceChangePassword`, `MemberOf`) menentukan taktik eskalsasi.
- BloodHound **bukan** exploit tool; ia memetakan jalur — eksekusi tetap pakai impacket/PowerView/certipy.
