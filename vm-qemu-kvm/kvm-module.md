Agar **KVM aktif saat boot** di Linux, biasanya yang perlu dipastikan adalah:

1. Modul kernel KVM dimuat otomatis saat sistem boot.
2. Layanan virtualisasi aktif.

Langkah-langkah:

1. Cek dukungan virtualisasi CPU

```
egrep -c '(vmx|svm)' /proc/cpuinfo
```

Jika hasilnya lebih dari `0`, berarti CPU mendukung virtualisasi.

2. Cek modul KVM yang diperlukan

Untuk Intel:

```
kvm
kvm_intel
```

Untuk AMD:

```
kvm
kvm_amd
```

3. Tambahkan modul agar dimuat saat boot

Edit atau buat file:

```
sudo nano /etc/modules-load.d/kvm.conf
```

Isi dengan:

Untuk Intel

```
kvm
kvm_intel
```

Untuk AMD

```
kvm
kvm_amd
```

4. Load modul tanpa reboot (opsional)

Intel

```
sudo modprobe kvm
sudo modprobe kvm_intel
```

AMD

```
sudo modprobe kvm
sudo modprobe kvm_amd
```

5. Aktifkan layanan libvirt saat boot

```
sudo systemctl enable libvirtd
sudo systemctl start libvirtd
```

6. Verifikasi modul sudah aktif

```
lsmod | grep kvm
```

Biasanya akan muncul seperti:

```
kvm_intel
kvm
```

7. Cek apakah KVM siap digunakan

Install tool:

```
sudo apt install cpu-checker
```

Lalu jalankan:

```
kvm-ok
```

Jika berhasil akan muncul:

```
KVM acceleration can be used
```
