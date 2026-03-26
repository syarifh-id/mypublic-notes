# Ringkasan Percakapan: Menghapus Windows dari Dual Boot dan Menggunakan SSD di Ubuntu

## 1. Kondisi Awal

Sistem menggunakan dual boot Ubuntu dan Windows yang terpasang pada dua
SSD berbeda.

Struktur disk: - `nvme0n1` : SSD berisi Ubuntu - `nvme0n1p1` →
`/boot/efi` - `nvme0n1p4` → `/` (root Ubuntu) - `nvme1n1` : SSD berisi
Windows

Partisi Windows memiliki pola khas: - Microsoft Reserved Partition (16
MB) - Partisi utama Windows (NTFS besar) - Recovery partition

## 2. Menghapus Windows dengan Aman

Karena Windows berada di SSD berbeda, seluruh partisi pada disk tersebut
dapat dihapus tanpa mempengaruhi Ubuntu.

Langkah umum: 1. Identifikasi disk dengan `lsblk` 2. Hapus partisi
Windows melalui GParted atau `fdisk` 3. Pastikan tidak menghapus partisi
Ubuntu (`nvme0n1`)

Bootloader Ubuntu tetap aman karena berada pada: `/boot/efi` di
`nvme0n1p1`.

## 3. Microsoft Reserved Partition

Partisi 16 MB yang terlihat sebagai **Microsoft Reserved Partition
(MSR)** dibuat otomatis oleh Windows pada disk GPT.

Karakteristik: - Tidak memiliki filesystem - Digunakan internal oleh
Windows - Tidak diperlukan oleh Linux

Jika masih ada di disk Ubuntu, partisi tersebut aman dibiarkan atau
dihapus karena ukurannya sangat kecil.

## 4. Masalah Saat Membuat Partisi Baru

Pesan berikut muncul saat menggunakan `fdisk`:

    No enough free sectors available

Penyebab: - Disk sudah memiliki partisi yang menggunakan seluruh ruang.

Contoh:

    nvme1n1
    └─nvme1n1p1 238.5G

Karena seluruh disk sudah menjadi satu partisi, partisi baru tidak bisa
dibuat.

## 5. Menyiapkan SSD Agar Bisa Dipakai di Ubuntu

### Format filesystem

    sudo mkfs.ext4 /dev/nvme1n1p1

### Membuat direktori mount

    sudo mkdir /mnt/ssd

### Mount SSD

    sudo mount /dev/nvme1n1p1 /mnt/ssd

## 6. Mengakses SSD

### Melalui terminal

    cd /mnt/ssd
    ls

### Menyimpan file

    cp file.txt /mnt/ssd

### Membuat folder

    mkdir /mnt/ssd/data

### Mengatasi permission

    sudo chown -R $USER:$USER /mnt/ssd

Setelah itu SSD dapat digunakan seperti direktori biasa di sistem
Ubuntu.

## 7. Mount Otomatis Saat Boot

Ambil UUID:

    sudo blkid /dev/nvme1n1p1

Edit `/etc/fstab`:

    UUID=UUID_PARTISI /mnt/ssd ext4 defaults 0 2

Uji konfigurasi:

    sudo mount -a

Jika tidak ada error, SSD akan otomatis ter-mount setiap sistem boot.
