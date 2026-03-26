# Ringkasan Port Forwarding KVM NAT di Ubuntu

## 1. Konsep Dasar
- VM KVM dengan network NAT (virbr0) hanya bisa:
  - VM → internet
  - Host → VM
- Perangkat lain di jaringan tidak bisa langsung akses VM
- Solusi: gunakan **port forwarding (DNAT)** di host

---

## 2. Port Forwarding dengan iptables

### Aktifkan IP Forwarding
```
sudo sysctl -w net.ipv4.ip_forward=1
```

### Rule NAT (DNAT)
Contoh:
Host:8902 → VM:6080

```
sudo iptables -t nat -A PREROUTING -p tcp --dport 8902 -j DNAT --to-destination 192.168.122.254:6080
```

### Izinkan Forwarding
```
sudo iptables -A FORWARD -p tcp -d 192.168.122.254 --dport 6080 -j ACCEPT
```

---

## 3. Firewall Host

### Jika pakai UFW
```
sudo ufw allow 8902/tcp
```

### Jika pakai iptables
```
sudo iptables -A INPUT -p tcp --dport 8902 -j ACCEPT
```

---

## 4. Perbedaan Listening vs NAT
- `ss -tunlp` hanya menampilkan service yang listening
- Port forwarding tidak muncul di ss karena hanya redirect
- Ini normal

---

## 5. Verifikasi NAT
Cek rule NAT:
```
sudo iptables -t nat -L -n -v | grep 8902
```

Contoh:
```
DNAT tcp -- 0.0.0.0/0 0.0.0.0/0 tcp dpt:8902 to:192.168.122.254:6080
```

---

## 6. Pengujian Koneksi

### Dari host
```
curl -k https://192.168.122.254:6080/vnc.html
```

### Dari jaringan lain
```
curl -k https://IP_HOST:8902/vnc.html
```

---

## 7. Masalah Umum

### 1. Rule FORWARD tidak efektif
Karena urutan chain:
- DOCKER
- LIBVIRT

Solusi:
```
sudo iptables -I FORWARD 1 -p tcp -d 192.168.122.254 --dport 6080 -j ACCEPT
```

---

### 2. IP VM salah
```
virsh domifaddr NAMA_VM
```

---

### 3. Service tidak bind ke semua interface
Harus:
```
0.0.0.0:PORT
```

---

### 4. HTTPS self-signed
Gunakan:
```
curl -k
```

---

## 8. Alur Trafik

```
Client
  ↓
Host:8902
  ↓ (DNAT)
VM:192.168.122.254:6080
  ↓
Service (noVNC)
```
