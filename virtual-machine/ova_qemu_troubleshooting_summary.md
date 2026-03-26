# Ringkasan Troubleshooting OVA → QEMU

## 1. Menjalankan OVA di QEMU

File `.ova` sebenarnya adalah arsip `tar` yang berisi: - `.ovf`
(deskripsi VM) - `.vmdk` (disk virtual) - `.mf` (manifest)

Langkah umum:

``` bash
tar -xvf vm.ova
qemu-img convert -f vmdk -O qcow2 disk.vmdk disk.qcow2
```

Menjalankan VM:

``` bash
qemu-system-x86_64 -m 4096 -smp 2 -hda disk.qcow2 -enable-kvm
```

Alternatif menggunakan `virt-install` untuk import ke libvirt.

------------------------------------------------------------------------

## 2. Membuat Bridge Network

Bridge digunakan agar VM berada pada jaringan yang sama dengan host.

Contoh:

``` bash
sudo ip link add name br0 type bridge
sudo ip link set br0 up
sudo ip link set eth0 master br0
```

Menjalankan QEMU dengan bridge:

``` bash
-netdev bridge,id=net0,br=br0
-device virtio-net-pci,netdev=net0
```

------------------------------------------------------------------------

## 3. Masalah Boot: Masuk GRUB lalu Initramfs

Penyebab umum: - Root filesystem tidak ditemukan - Perubahan nama disk
(`sda` → `vda`) - UUID tidak cocok - Driver disk tidak ada di initramfs

Diagnosis dari initramfs:

``` bash
blkid
ls /dev
cat /proc/cmdline
```

Solusi sementara: ubah parameter kernel di GRUB:

    root=/dev/sda1
    atau
    root=/dev/vda1

------------------------------------------------------------------------

## 4. Disk Tidak Terdeteksi

Jika `blkid` tidak menampilkan apa pun berarti kernel tidak mendeteksi
disk.

Penyebab: perbedaan controller disk antara VMware/VirtualBox dan QEMU.

Solusi sementara menjalankan VM dengan controller lama:

``` bash
qemu-system-x86_64 -drive file=disk.qcow2,format=qcow2,if=ide
```

atau SATA/AHCI.

------------------------------------------------------------------------

## 5. Boot Sangat Lama

Penyebab umum: - menggunakan IDE (lambat) - kernel mencoba mencari root
filesystem - fsck berjalan - timeout DHCP

------------------------------------------------------------------------

## 6. Layar Blank dengan Kursor

Kernel sudah boot tetapi display manager bermasalah.

Langkah diagnosis: - pindah ke TTY

    Ctrl + Alt + F2

-   cek log:

``` bash
journalctl -xb
```

Solusi lain: boot dengan `nomodeset` atau gunakan VGA standar:

``` bash
-vga std
```

------------------------------------------------------------------------

## 7. Modul Virtio Tidak Lengkap

Hasil:

    virtio
    virtio_ring

Modul penting belum dimuat: - virtio_pci - virtio_blk - virtio_net

Load manual:

``` bash
modprobe virtio_pci
modprobe virtio_blk
modprobe virtio_net
```

Tambahkan ke initramfs:

    /etc/initramfs-tools/modules

Isi:

    virtio_pci
    virtio_blk
    virtio_net

Lalu rebuild:

``` bash
update-initramfs -u
update-grub
```

------------------------------------------------------------------------

## 8. VM Terasa Berat / Lag

Penyebab utama:

1.  Tidak menggunakan akselerasi KVM
2.  Controller disk IDE lambat
3.  VGA emulasi lambat
4.  CPU core sedikit

Solusi menjalankan VM dengan akselerasi:

``` bash
qemu-system-x86_64 -enable-kvm -cpu host -m 4096 -smp 2
```

Setelah driver virtio tersedia, jalankan disk dengan:

``` bash
-drive file=disk.qcow2,format=qcow2,if=virtio
```

------------------------------------------------------------------------

## 9. Alur Troubleshooting yang Disarankan

1.  Extract OVA
2.  Convert VMDK → QCOW2
3.  Boot dengan IDE/SATA agar sistem bisa masuk
4.  Perbaiki initramfs dan tambahkan driver virtio
5.  Rebuild initramfs
6.  Jalankan kembali VM dengan virtio + KVM untuk performa optimal
