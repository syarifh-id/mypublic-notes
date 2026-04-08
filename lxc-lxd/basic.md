Berikut dasar penggunaan **LXC** dan **LXD** secara praktis dan ringkas.

---

# 1. Konsep Dasar

## LXC (Linux Containers)

* Low-level container runtime (mirip seperti “library”).
* Memberikan API untuk membuat dan menjalankan container Linux.
* Lebih manual dan kompleks.

## LXD

* High-level manager di atas LXC.
* Lebih mudah digunakan (seperti “Docker tapi system container”).
* Mendukung REST API, CLI, image management, network, storage.

Singkatnya:

* LXC = engine
* LXD = management layer

---

# 2. Instalasi (Ubuntu)

```bash
sudo apt update
sudo apt install lxd
```

Tambahkan user ke group:

```bash
sudo usermod -aG lxd $USER
newgrp lxd
```

Inisialisasi:

```bash
lxd init
```

Pilih default saja jika belum paham (storage: dir, network: auto).

---

# 3. Perintah Dasar LXD

## 3.1 List container

```bash
lxc list
```

---

## 3.2 Membuat container

```bash
lxc launch ubuntu:22.04 my-container
```

Keterangan:

* `ubuntu:22.04` = image
* `my-container` = nama container

---

## 3.3 Masuk ke container

```bash
lxc exec my-container -- bash
```

---

## 3.4 Stop & start

```bash
lxc stop my-container
lxc start my-container
```

---

## 3.5 Hapus container

```bash
lxc delete my-container
```

Force delete:

```bash
lxc delete my-container --force
```

---

# 4. Image Management

## Cari image

```bash
lxc image list images:
```

## Gunakan image lain

```bash
lxc launch images:debian/12 debian-container
```

---

# 5. Networking

LXD biasanya otomatis buat bridge `lxdbr0`.

Cek:

```bash
lxc network list
```

Lihat IP container:

```bash
lxc list
```

---

# 6. File Transfer

## Copy file ke container

```bash
lxc file push file.txt my-container/root/
```

## Copy dari container

```bash
lxc file pull my-container/root/file.txt .
```

---

# 7. Snapshot

```bash
lxc snapshot my-container snap1
```

Restore:

```bash
lxc restore my-container snap1
```

---

# 8. Exec Command Tanpa Masuk Shell

```bash
lxc exec my-container -- apt update
```

---

# 9. Resource Limit

```bash
lxc config set my-container limits.cpu 2
lxc config set my-container limits.memory 1GB
```

---

# 10. Bedanya dengan Docker

| Aspek       | LXD              | Docker                |
| ----------- | ---------------- | --------------------- |
| Tipe        | System container | Application container |
| Init system | Bisa (systemd)   | Biasanya tidak        |
| Use case    | VM ringan        | Microservices         |
| OS penuh    | Ya               | Tidak                 |

---

# 11. Use Case Umum

* Lab penetration testing (multi target)
* Simulasi jaringan
* Hosting service ringan
* CI/CD environment
* Sandbox testing exploit

---

# 12. Struktur Penting

Lokasi data:

```bash
/var/snap/lxd/common/lxd/
```

---

Kalau mau lanjut, bisa masuk ke:

* setup bridge manual (advanced networking)
* expose service container ke host
* clustering LXD
* atau integrasi dengan tools pentest lab (sesuai pekerjaan kamu)
