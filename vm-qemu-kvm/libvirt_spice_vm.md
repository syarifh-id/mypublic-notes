# Ringkasan Percakapan: Manajemen VM dengan libvirt, virsh, VNC, dan SPICE

## 1. Dasar Penggunaan virsh

-   Melihat daftar VM: `virsh list --all`
-   Start / stop VM: `virsh start`, `virsh shutdown`, `virsh destroy`
-   Edit konfigurasi VM: `virsh edit nama_vm`
-   Melihat XML VM: `virsh dumpxml nama_vm`
-   Console CLI: `virsh console nama_vm`

------------------------------------------------------------------------

## 2. Storage Pool

-   Melihat pool: `virsh pool-list --all`
-   Stop pool: `virsh pool-destroy nama_pool`
-   Hapus definisi: `virsh pool-undefine nama_pool`
-   Refresh pool: `virsh pool-refresh nama_pool`
-   Lokasi pool bisa di luar `/var/lib/libvirt/`
-   Konfigurasi pool disimpan di:
    -   `/etc/libvirt/storage/`

------------------------------------------------------------------------

## 3. Lokasi Konfigurasi Penting

-   Konfigurasi VM: `/etc/libvirt/qemu/`
-   Snapshot metadata: `/etc/libvirt/qemu/snapshot/`
-   Log VM: `/var/log/libvirt/qemu/`
-   Disk VM: sesuai lokasi storage pool

------------------------------------------------------------------------
### install via virt-install
```bash
virt-install --name proxmox --ram 8192 --vcpus 4 --cpu host --disk path=~/Tools/VMS/proxmox.qcow2,size=64 --cdrom ~/Tools/proxmox-ve_9.1-1.iso --network network=default --graphics spice --osinfo debian12 --cpu host-passthrough
```



## 4. VNC vs SPICE

### VNC

-   Cek display: `virsh domdisplay nama_vm`
-   Format: `vnc://127.0.0.1:1`
-   Port = 5900 + display number
-   Tidak mendukung shared clipboard

### SPICE

-   Konfigurasi XML:

    ``` xml
    <graphics type='spice' autoport='yes' listen='127.0.0.1'/>
    ```

-   Lebih responsif

-   Mendukung shared clipboard

-   Harus menggunakan `virt-viewer` atau console bawaan virt-manager

------------------------------------------------------------------------

## 5. Membuka GUI VM

-   Buka console langsung:

    ``` bash
    virt-viewer nama_vm
    ```

-   Remote via SSH:

    ``` bash
    virt-viewer --connect qemu+ssh://user@host/system nama_vm
    ```

-   Jika terbuka dua window:

    -   Cek `heads='1'`
    -   Pastikan hanya satu `<graphics>`
    -   Pastikan tidak ada dua `<video>`
    -   Gunakan `virt-viewer --attach`

------------------------------------------------------------------------

## 6. Error SPICE Channel Duplicate

Error:

    virtio-serial-bus: A port already exists by name com.redhat.spice.0

Solusi: - Pastikan hanya satu:
`xml   <channel type='spicevmc'>     <target type='virtio' name='com.redhat.spice.0'/>   </channel>`

------------------------------------------------------------------------

## 7. Shared Clipboard SPICE

Syarat agar aktif: - Backend = SPICE - Channel `com.redhat.spice.0`
ada - `spice-vdagent` aktif di guest - Viewer menggunakan SPICE

Install di Kali:

``` bash
sudo apt install spice-vdagent
sudo systemctl enable spice-vdagent
sudo systemctl start spice-vdagent
```

------------------------------------------------------------------------

## 8. Masalah di i3 Window Manager

Masalah: - Error "referenced but unset environment variable" - Channel
tetap disconnected

Penyebab: - i3 tidak otomatis menjalankan DBus session

Solusi: - Jalankan i3 via: `bash   exec dbus-run-session i3` - Tambahkan
ke config i3: `bash   exec_always --no-startup-id spice-vdagent` -
Pastikan: `bash   echo $DISPLAY   echo $DBUS_SESSION_BUS_ADDRESS`

------------------------------------------------------------------------

## 9. Checklist Stabil SPICE + i3

1.  Backend = SPICE
2.  Hanya satu `<graphics>`
3.  Hanya satu `<channel spicevmc>`
4.  `virtio-serial` controller ada
5.  `spice-vdagent` aktif
6.  Login ke desktop GUI
7.  Gunakan `virt-viewer` atau console virt-manager

------------------------------------------------------------------------

Dokumen ini merangkum seluruh proses troubleshooting dan konfigurasi
terkait virsh, storage pool, VNC, SPICE, shared clipboard, serta
penyesuaian pada i3 window manager.


Jika menggunakan **CLI untuk meng-import OVA ke lingkungan yang dipakai oleh `virt-manager`**, biasanya menggunakan tool dari **libvirt**, yaitu **`virt-install`**.

Langkah umum:

1. Ekstrak file OVA

```bash
tar -xvf vm.ova
```

Biasanya menghasilkan:

```
vm.ovf
disk.vmdk
```

2. Konversi disk ke qcow2 (opsional tetapi umum digunakan)

```bash
qemu-img convert -f vmdk -O qcow2 disk.vmdk disk.qcow2
```

3. Import VM menggunakan CLI

```bash
virt-install \
--name vm-name \
--memory 4096 \
--vcpus 2 \
--disk path=disk.qcow2,format=qcow2 \
--import \
--os-variant generic \
--network network=default \
--graphics spice
```

Penjelasan parameter:

* `--name` nama VM
* `--memory` RAM (MB)
* `--vcpus` jumlah CPU
* `--disk` lokasi disk
* `--import` menandakan VM sudah memiliki OS
* `--os-variant` tipe OS
* `--network network=default` NAT libvirt
* `--graphics spice` agar bisa dibuka dari virt-manager

4. Mengecek VM yang sudah dibuat

```bash
virsh list --all
```

5. Menjalankan VM

```bash
virsh start vm-name
```

6. Menghubungkan console

```bash
virt-viewer vm-name
```

Pendekatan ini membuat VM langsung muncul di **virt-manager** karena menggunakan backend **libvirt** yang sama.
