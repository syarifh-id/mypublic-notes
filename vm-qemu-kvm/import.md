````md
# Ringkasan Percakapan: Import QCOW2 ke KVM dan Pembersihan VirtualBox

## 1. Import QCOW2 dengan virt-install

Perintah dasar untuk meng-import disk QCOW2 yang sudah berisi sistem menggunakan `virt-install`:

```bash
virt-install \
--name vm-name \
--memory 4096 \
--vcpus 2 \
--cpu host-passthrough \
--disk path=/var/lib/libvirt/images/vm.qcow2,format=qcow2,bus=virtio \
--network network=default,model=virtio \
--graphics spice \
--os-variant generic \
--import
````

Parameter penting:

* `--import` → boot langsung dari disk yang sudah ada
* `bus=virtio` → performa disk lebih baik
* `--cpu host-passthrough` → performa CPU maksimal
* `model=virtio` → network lebih cepat

---

# 2. Masalah Boot Setelah Import

Contoh pesan boot:

```
/dev/vda3: clean ...
[FAILED] Failed to start isc-dhcp-server.service
```

Makna:

* filesystem berhasil dibaca
* kernel berhasil mount disk
* error hanya pada service DHCP

Kemungkinan penyebab:

* interface jaringan berubah nama
* konfigurasi DHCP menunjuk interface lama

Cek interface:

```bash
ip a
```

Perbaiki konfigurasi:

```bash
sudo nano /etc/default/isc-dhcp-server
```

---

# 3. Layar Blank Saat Boot

Layar kosong beberapa saat setelah boot bisa terjadi karena:

1. systemd menunggu timeout service
2. deteksi hardware baru
3. konfigurasi network tidak cocok
4. console login berpindah tty

Tes:

```
Enter
Ctrl + Alt + F2
```

---

# 4. Penyebab VM Sangat Lambat Setelah Import

Beberapa penyebab utama:

### Disk bus tidak virtio

Cek:

```bash
virsh dumpxml vm-name | grep bus
```

Ideal:

```xml
<target dev='vda' bus='virtio'/>
```

---

### CPU bukan host

Ideal:

```xml
<cpu mode='host-passthrough'/>
```

---

### KVM tidak aktif

Cek:

```bash
lsmod | grep kvm
```

Harus muncul:

```
kvm
kvm_intel
```

atau

```
kvm_amd
```

---

### Disk cache tidak optimal

Rekomendasi:

```xml
<driver name='qemu' type='qcow2' cache='none' io='native'/>
```

---

# 5. Sisa VirtualBox Guest Additions

Pesan boot yang muncul:

```
vboxadd.service
vboxadd-service.service
```

Ini berasal dari Guest Additions VirtualBox yang masih ada di sistem.

---

# 6. Menghapus VirtualBox dari Sistem

## Hapus paket

```bash
sudo apt purge virtualbox* virtualbox-guest* virtualbox-ext-pack
sudo apt autoremove --purge
```

---

## Disable service

```bash
sudo systemctl disable vboxadd.service
sudo systemctl disable vboxadd-service.service
sudo systemctl disable vboxadd-x11.service
```

---

## Hapus file service

```bash
sudo rm /lib/systemd/system/vboxadd.service
sudo rm /lib/systemd/system/vboxadd-service.service
sudo rm /lib/systemd/system/vboxadd-x11.service
```

atau

```bash
sudo rm /etc/systemd/system/vboxadd*
```

---

## Reload systemd

```bash
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

---

## Hapus modul kernel

```bash
sudo rm -rf /lib/modules/$(uname -r)/misc/vbox*
sudo depmod -a
```

---

## Hapus konfigurasi

```bash
rm -rf ~/.config/VirtualBox
rm -rf ~/.VirtualBox
sudo rm -rf /usr/lib/virtualbox
sudo rm -rf /etc/vbox
```

---

## Verifikasi

```bash
lsmod | grep vbox
systemctl list-unit-files | grep vbox
```

Jika tidak ada output, sistem sudah bersih dari VirtualBox.

```
```
