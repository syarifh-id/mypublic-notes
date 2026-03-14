# Proxmox Nested Virtualization Lab Conversation

## 1. Install Proxmox di Ubuntu dengan KVM-QEMU

User ingin menjalankan Proxmox sebagai VM di Ubuntu menggunakan KVM/QEMU
(nested virtualization).

Langkah utama: - Install qemu-kvm, libvirt, virt-manager - Aktifkan
nested virtualization (Intel: kvm_intel nested=1, AMD: kvm_amd
nested=1) - Gunakan virt-install dengan --cpu host-passthrough - Gunakan
--osinfo debian12 - Gunakan network=default jika bridge br0 belum ada

------------------------------------------------------------------------

## 2. Error --os-variant / --osinfo Required

Solusi: Gunakan parameter: --osinfo debian12

Atau: --osinfo linux2022

Contoh perintah: 
```
virt-install --name proxmox --ram 8192 --vcpus 4 --cpu host-passthrough --disk path=/var/lib/libvirt/images/proxmox.qcow2,size=64,format=qcow2 --cdrom /path/to/proxmox.iso --network network=default --graphics vnc --osinfo debian12
```
------------------------------------------------------------------------

## 3. Error Cannot get interface MTU on 'br0'

Penyebab: Bridge br0 belum dibuat.

Solusi cepat: Gunakan: --network network=default

Cek network: virsh net-list --all

------------------------------------------------------------------------

## 4. Membuat VM dan LXC di Proxmox

### VM (KVM)

-   Upload ISO
-   Create VM
-   CPU: host
-   Disk: VirtIO (atau SATA jika perlu)
-   Network: vmbr0

### LXC

-   Download CT Template
-   Create CT
-   Set CPU, RAM, Disk
-   Bridge: vmbr0

------------------------------------------------------------------------

## 5. Kernel Panic Saat Boot VM

Kemungkinan penyebab: - Nested belum aktif - CPU type bukan host - RAM
terlalu kecil - Disk bus tidak cocok - VirtIO tanpa driver

Solusi: - Gunakan CPU type: host atau host-passthrough - Tambah RAM -
Ubah disk ke SATA jika perlu

------------------------------------------------------------------------

## 6. CPU Type x86-64-v2-AES

Tidak ideal untuk nested virtualization. Rekomendasi: Gunakan host atau
host-passthrough.

------------------------------------------------------------------------

## 7. Cara Akses VNC VM

Cek port VNC: virsh vncdisplay proxmox

Akses via: remote-viewer vnc://127.0.0.1:5900

Untuk VM dalam Proxmox: Akses melalui web: https://IP_PROXMOX:8006

------------------------------------------------------------------------

## 8. Simulasi User Remote

Akses dari LAN: https://IP_PROXMOX:8006

Jika nested NAT: Gunakan port forwarding atau bridge network.

Untuk akses internet: Gunakan VPN atau SSH tunnel.

------------------------------------------------------------------------

## 9. User Hanya Akses VNC

Metode disarankan: Buat role khusus dengan privilege: - VM.Console -
VM.Audit

Assign permission ke VM tertentu saja.

Alternatif (tidak aman): Expose raw VNC port 590x langsung.

Lebih aman: Gunakan SSH tunnel untuk VNC.

------------------------------------------------------------------------

End of conversation transcript.
