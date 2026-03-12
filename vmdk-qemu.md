Pada struktur tersebut, file yang menjadi **descriptor utama** adalah:

`Rocky Linux 8.vmdk`

Sedangkan file `-s001.vmdk`, `-s002.vmdk`, dan seterusnya adalah **split disk parts** dari disk utama. QEMU akan otomatis membaca semuanya jika menggunakan file descriptor utama.

Langkah konversi:

1. Masuk ke direktori VM

```
cd "SELinux Lab VM"
```

2. Konversi VMDK ke QCOW2

Karena nama file mengandung spasi, gunakan tanda kutip.

```
qemu-img convert -f vmdk -O qcow2 "Rocky Linux 8.vmdk" rocky8.qcow2
```

Proses ini akan menggabungkan seluruh file:

```
Rocky Linux 8-s001.vmdk
Rocky Linux 8-s002.vmdk
...
Rocky Linux 8-s011.vmdk
```

menjadi satu file `rocky8.qcow2`.

3. Cek hasil disk

```
qemu-img info rocky8.qcow2
```

4. Jalankan di QEMU/KVM

Contoh:

```
qemu-system-x86_64 \
-m 4G \
-smp 4 \
-enable-kvm \
-drive file=rocky8.qcow2,format=qcow2
```

Jika ingin konversi lebih cepat bisa menggunakan opsi paralel dan preallocation:

```
qemu-img convert -p -f vmdk -O qcow2 "Rocky Linux 8.vmdk" rocky8.qcow2
```

Jika muncul error seperti **"Invalid VMDK image descriptor"**, biasanya descriptor `.vmdk` rusak dan perlu diperbaiki atau dikonversi dari file `-s001.vmdk`.
