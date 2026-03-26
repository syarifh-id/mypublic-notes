# Ringkasan Percakapan: Import QCOW2 dan Optimasi VM KVM

## 1. Import QCOW2 Menggunakan virt-install

Untuk mengimport disk QCOW2 yang sudah berisi sistem operasi, gunakan
mode `--import` agar tidak menjalankan installer ulang.

Contoh:

    virt-install --name vm-name --ram 4096 --vcpus 2 --disk path=/path/to/disk.qcow2,format=qcow2,bus=virtio --os-variant generic --network network=default,model=virtio --graphics spice --import

Parameter penting: - `--name` : nama VM - `--ram` : jumlah RAM -
`--vcpus` : jumlah CPU - `--disk` : lokasi disk QCOW2 - `bus=virtio` :
performa disk lebih baik - `--network` : konfigurasi jaringan -
`--import` : langsung boot dari disk yang ada

------------------------------------------------------------------------

## 2. Masalah Boot Setelah Import

Beberapa masalah yang muncul saat boot:

### Journal Recovery

Pesan:

    /dev/vda3: clean ...

Artinya filesystem berhasil dibaca dan disk tidak rusak.

### Service DHCP Gagal

Pesan:

    Failed to start isc-dhcp-server.service

Penyebab umum: - nama interface jaringan berubah (misalnya `eth0`
menjadi `ens3`) - konfigurasi DHCP masih menunjuk interface lama

Solusi: 1. cek interface

    ip a

2.  ubah konfigurasi

```{=html}
<!-- -->
```
    /etc/default/isc-dhcp-server

------------------------------------------------------------------------

## 3. Layar Blank Saat Boot

Layar kosong beberapa saat bisa terjadi karena: - systemd menunggu
timeout service - deteksi hardware baru - konfigurasi jaringan - console
berpindah

Untuk memunculkan login:

    Enter

atau

    Ctrl + Alt + F2

------------------------------------------------------------------------

## 4. Menghapus Sisa VirtualBox Guest Additions

Jika VM berasal dari VirtualBox, sering masih ada service:

-   `vboxadd.service`
-   `vboxadd-service.service`
-   `vboxadd-x11.service`

### Menghapus paket

    sudo apt purge virtualbox-guest-utils virtualbox-guest-x11 virtualbox-guest-dkms

### Disable service

    sudo systemctl disable vboxadd.service
    sudo systemctl disable vboxadd-service.service
    sudo systemctl disable vboxadd-x11.service

### Hapus file service

    sudo rm /lib/systemd/system/vboxadd*
    sudo rm /etc/systemd/system/vboxadd*

Reload systemd:

    sudo systemctl daemon-reload

------------------------------------------------------------------------

## 5. Penyebab VM Hasil Import Lambat

Beberapa penyebab utama:

### Disk bus bukan virtio

Periksa:

    virsh dumpxml vm-name | grep bus

Ideal:

    <target dev='vda' bus='virtio'/>

### CPU tidak menggunakan host

Gunakan:

    --cpu host-passthrough

### KVM acceleration tidak aktif

Periksa:

    lsmod | grep kvm

### Cache disk tidak optimal

Konfigurasi ideal:

    <driver name='qemu' type='qcow2' cache='none' io='native'/>

### Image QCOW2 belum dioptimasi

Optimasi:

    qemu-img convert -O qcow2 source.qcow2 optimized.qcow2

------------------------------------------------------------------------

## 6. Membuat VM Import Dengan CPU Host

Contoh konfigurasi lengkap:

    virt-install --name vm-name --memory 4096 --vcpus 4 --cpu host-passthrough --disk path=/var/lib/libvirt/images/vm.qcow2,format=qcow2,bus=virtio --network network=default,model=virtio --graphics spice --os-variant generic --import

Keuntungan: - fitur CPU host langsung digunakan - performa VM lebih baik

------------------------------------------------------------------------

## 7. Menghapus Service VirtualBox Yang Sudah Disable

Disable hanya menghentikan autostart, bukan menghapus file.

Cari lokasi service:

    systemctl cat vboxadd.service

Hapus file:

    sudo rm /lib/systemd/system/vboxadd.service
    sudo rm /lib/systemd/system/vboxadd-service.service
    sudo rm /lib/systemd/system/vboxadd-x11.service

Reload systemd:

    sudo systemctl daemon-reload

Reset status gagal:

    sudo systemctl reset-failed

Cek kembali:

    systemctl list-unit-files | grep vbox

------------------------------------------------------------------------

Dokumen ini merangkum pembahasan mengenai: - import VM QCOW2 -
troubleshooting boot - pembersihan VirtualBox Guest Additions - optimasi
performa VM pada KVM
