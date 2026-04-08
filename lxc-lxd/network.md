Berikut penjelasan **setup bridge manual (advanced networking) di LXD** yang biasanya dipakai untuk kebutuhan lab, pentest, atau integrasi langsung ke jaringan LAN.

---

# 1. Konsep

Secara default, LXD membuat bridge internal (`lxdbr0`) dengan NAT.

Kalau pakai **bridge manual**:

* Container dapat IP dari jaringan LAN
* Bisa diakses langsung seperti device biasa
* Tidak melalui NAT

Topologi sederhana:

```
[ Router ] --- [ eth0 host ] --- [ br0 ] --- [ container ]
```

---

# 2. Buat Linux Bridge di Host

## Install bridge-utils (optional)

```bash
sudo apt install bridge-utils
```

## Buat bridge (temporary)

```bash
sudo ip link add br0 type bridge
sudo ip link set br0 up
```

## Tambahkan interface fisik

Misal interface kamu `eth0`:

```bash
sudo ip link set eth0 master br0
```

---

## Konfigurasi IP ke bridge (bukan ke eth0)

```bash
sudo dhclient br0
```

PENTING:

* Jangan assign IP ke `eth0` lagi
* Semua pindah ke `br0`

---

# 3. Konfigurasi Permanen (Netplan - Ubuntu)

Edit:

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

Contoh:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no

  bridges:
    br0:
      interfaces: [eth0]
      dhcp4: yes
```

Apply:

```bash
sudo netplan apply
```

---

# 4. Hubungkan Bridge ke LXD

## Buat network baru di LXD

```bash
lxc network create br0-network \
  bridge.external_interfaces=br0 \
  ipv4.address=none \
  ipv6.address=none
```

Penjelasan:

* `external_interfaces=br0` → pakai bridge host
* `ipv4.address=none` → tidak pakai NAT LXD

---

# 5. Attach ke Container

## Launch container dengan network baru

```bash
lxc launch ubuntu:22.04 c1 -n br0-network
```

Atau attach ke container existing:

```bash
lxc network attach br0-network c1 eth0
```

---

# 6. Cek IP Container

Masuk ke container:

```bash
lxc exec c1 -- ip a
```

Biasanya:

* Akan dapat IP dari DHCP router
* Satu subnet dengan host

---

# 7. Verifikasi

Dari host:

```bash
ping <IP-container>
```

Dari device lain:

```bash
ping <IP-container>
```

Kalau berhasil → container sudah full LAN member.

---

# 8. Mode Lain (Alternatif Advanced)

## Macvlan (lebih simpel, tanpa bridge manual)

```bash
lxc network create macvlan0 \
  parent=eth0 \
  type=macvlan
```

Launch:

```bash
lxc launch ubuntu:22.04 c2 -n macvlan0
```

Kelebihan:

* Lebih cepat setup

Kekurangan:

* Host tidak bisa komunikasi ke container

---

# 9. Perbandingan

| Mode          | Host ↔ Container | LAN Access | Kompleksitas |
| ------------- | ---------------- | ---------- | ------------ |
| lxdbr0 (NAT)  | Ya               | via NAT    | Mudah        |
| bridge manual | Ya               | Full LAN   | Medium       |
| macvlan       | Tidak            | Full LAN   | Mudah        |

---

# 10. Use Case Nyata (sesuai security lab)

* Simulasi jaringan perusahaan
* MITM attack testing
* DHCP spoofing
* IDS/IPS testing
* Multi-host scanning tanpa NAT

---

# 11. Pitfall Umum

1. Host kehilangan koneksi
   → karena IP masih di `eth0`, bukan `br0`

2. Container tidak dapat IP
   → DHCP router tidak jalan / VLAN issue

3. WiFi tidak bisa bridge
   → sebagian besar interface wireless tidak support bridge layer 2

---

Kalau ingin lebih dalam lagi, bisa lanjut ke:

* VLAN tagging di LXD bridge
* multiple NIC per container (dual network)
* integrasi dengan tools seperti bettercap / mitmproxy dalam lab jaringan


Dari output yang kamu kirim, kondisi sekarang:

* Container `mycontainer` hanya dapat **IPv6**
* Tidak dapat IPv4
* Terhubung ke `lxdbr0` (managed NAT network)
* `lxdbr0` punya IPv4 (`10.10.205.1/24`), tapi container tidak dapat DHCP IPv4

Artinya ada kemungkinan:

* DHCP IPv4 di `lxdbr0` tidak jalan
* atau config container/network bermasalah

---

# 1. Cek DHCP LXD

Cek config network:

```bash
lxc network show lxdbr0
```

Pastikan ada:

```yaml
ipv4.address: 10.10.205.1/24
ipv4.nat: "true"
ipv4.dhcp: "true"
```

Kalau `ipv4.dhcp` tidak ada / false → itu penyebabnya.

---

# 2. Fix DHCP IPv4

Set ulang:

```bash
lxc network set lxdbr0 ipv4.dhcp true
lxc network set lxdbr0 ipv4.nat true
```

Restart container:

```bash
lxc restart mycontainer
```

---

# 3. Cek dari dalam container

```bash
lxc exec mycontainer -- ip a
```

Kalau masih belum ada IPv4:

```bash
lxc exec mycontainer -- dhclient eth0
```

---

# 4. Kemungkinan lain (yang sering terjadi)

## A. Image tidak jalan DHCP client

Beberapa image minimal (terutama alpine/debian minimal) tidak auto DHCP.

Cek:

```bash
lxc exec mycontainer -- ps aux | grep dh
```

Solusi:

```bash
lxc exec mycontainer -- apt update
lxc exec mycontainer -- apt install isc-dhcp-client -y
```

---

## B. Firewall host blocking

Cek iptables / nftables:

```bash
sudo iptables -L
```

Kadang DHCP (UDP 67/68) ke-block.

---

# 5. Kalau tujuan kamu: Advanced Bridge (LAN langsung)

Dari network list:

* `enp8s0` → kandidat LAN
* `wlp9s0` → wifi (tidak bisa bridge normal)

Kalau kamu pakai **LAN (enp8s0)**, bisa lanjut ke bridge manual:

---

# 6. Setup cepat bridge dari kondisi kamu

## Step 1: buat bridge

```bash
sudo ip link add br0 type bridge
sudo ip link set br0 up
sudo ip link set enp8s0 master br0
sudo dhclient br0
```

---

## Step 2: buat network di LXD

```bash
lxc network create br0net \
  bridge.external_interfaces=br0 \
  ipv4.address=none \
  ipv6.address=none
```

---

## Step 3: attach ke container

```bash
lxc network attach br0net mycontainer eth1
```

Masuk container:

```bash
lxc exec mycontainer -- ip a
```

Harusnya:

* `eth1` dapat IP dari router LAN

---

# 7. Hal penting dari kondisi kamu

## Kamu punya 2 opsi:

### Opsi 1 (fix existing)

* Perbaiki `lxdbr0`
* Gunakan NAT (lebih stabil)

### Opsi 2 (advanced)

* Gunakan `br0`
* Container jadi 1 jaringan dengan LAN

---

# 8. Insight penting (yang sering bikin bingung)

Dari output kamu:

```text
lxdbr0 USED BY: 2
```

Artinya:

* Sudah ada 2 NIC terhubung
* Tapi DHCP IPv4 tetap gagal → indikasi config, bukan attach issue

---

# 9. Debug cepat (recommended urutan)

1. `lxc network show lxdbr0`
2. pastikan `ipv4.dhcp=true`
3. restart container
4. cek `ip a`
5. paksa `dhclient`

---

Kalau kamu ingin, bisa lanjut ke:

* kenapa DHCP LXD bisa gagal secara internal (dnsmasq issue)
* atau desain topologi lab pentest multi-segment pakai LXD bridge + VLAN


Dari config `lxdbr0` yang kamu kirim, terlihat jelas penyebabnya:

```yaml
ipv4.address: 10.10.205.1/24
ipv4.nat: "true"
# TIDAK ADA ipv4.dhcp
```

Artinya:

* Network sudah punya IPv4 subnet
* NAT aktif
* **Tapi DHCP IPv4 tidak diaktifkan**

Itu sebabnya container hanya dapat IPv6.

---

# 1. Aktifkan DHCP IPv4

Jalankan:

```bash
lxc network set lxdbr0 ipv4.dhcp true
```

Opsional (biasanya default, tapi aman diset):

```bash
lxc network set lxdbr0 ipv4.dhcp.stateful true
```

---

# 2. Restart Container

```bash
lxc restart mycontainer
```

---

# 3. Verifikasi

```bash
lxc list
```

Harusnya sekarang muncul IPv4 seperti:

```
10.10.205.x
```

Atau cek dari dalam:

```bash
lxc exec mycontainer -- ip a
```

---

# 4. Kalau masih belum dapat IPv4

Lanjut debug:

## Paksa DHCP manual

```bash
lxc exec mycontainer -- dhclient eth0
```

---

## Cek dnsmasq (DHCP server LXD)

LXD pakai dnsmasq di background:

```bash
ps aux | grep dnsmasq
```

Harus ada yang bind ke `lxdbr0`.

---

## Restart LXD

```bash
sudo systemctl restart snap.lxd.daemon
```

---

# 5. Insight penting

Config kamu sebenarnya sudah hampir benar, hanya kurang satu flag:

* `ipv4.address` → OK
* `ipv4.nat` → OK
* `ipv4.dhcp` → MISSING → ini kunci

---

# 6. Kenapa IPv6 tetap jalan?

Karena:

```yaml
ipv6.address: fd42:...
```

LXD otomatis aktifkan SLAAC (IPv6 autoconfig), jadi:

* IPv6 tidak butuh DHCP seperti IPv4
* langsung dapat IP

---

# 7. Rekomendasi untuk lab kamu

Kalau fokus ke:

* pentest tools
* scanning
* exploit

Lebih stabil pakai:

* NAT (`lxdbr0`) → untuk internet access
* bridge manual (`br0`) → untuk simulasi LAN

Bisa juga dual NIC:

* `eth0` → NAT (internet)
* `eth1` → LAN (target environment)

---

Kalau masih tidak muncul IPv4 setelah langkah ini, berarti masuk ke kasus:

* DHCP container issue
* firewall host
* atau dnsmasq LXD crash

dan itu bisa ditrace lebih dalam.
