# Troubleshooting QEMU/KVM SPICE + OpenGL (virgl)

## Ringkasan Masalah

VM tidak bisa diakses menggunakan virt-viewer dengan error: - Failed to
connect: Display can only be attached through libvirt with --attach - No
graphical display found - opengl is not available

## Analisis Awal

Konfigurasi XML: - SPICE aktif - OpenGL (virgl) diaktifkan - Rendernode
tersedia (/dev/dri/renderD128)

Namun: - SPICE tidak muncul saat runtime - VM gagal start saat GL aktif

## Investigasi

### 1. SPICE tidak tersedia

Perintah:

    virsh domdisplay kali-linux

Output:

    No graphical display found

Kesimpulan: SPICE gagal dibuat saat VM start.

------------------------------------------------------------------------

### 2. Error utama

    opengl is not available

Makna: QEMU gagal menginisialisasi OpenGL backend (EGL).

------------------------------------------------------------------------

### 3. Validasi lingkungan

#### GPU

    glxinfo | grep "OpenGL renderer"

Hasil: Mesa Intel UHD Graphics → OK

#### Device

    /dev/dri/renderD128

Ada dan accessible → OK

#### Permission

libvirt-qemu sudah masuk grup render → OK

#### QEMU support GL

    qemu-system-x86_64 -device help | grep gl

Hasil: - virtio-vga-gl - virtio-gpu-gl

→ QEMU support virgl → OK

------------------------------------------------------------------------

### 4. Test manual OpenGL

    qemu-system-x86_64 -display gtk,gl=on

Hasil: Berhasil membuka jendela

Kesimpulan: OpenGL + EGL di sistem berfungsi

------------------------------------------------------------------------

## Akar Masalah

Masalah bukan pada: - GPU - Permission - QEMU support - Library virgl

Masalah ada pada: - Libvirt (mode system) tidak bisa menginisialisasi
OpenGL runtime

------------------------------------------------------------------------

## Masalah tambahan

Walaupun GL dinonaktifkan:

    <gl enable='no'/>

VM tetap gagal karena:

    virtio-vga-gl

Penyebab:

    <acceleration accel3d='yes'/>

Masih memaksa penggunaan device GL

------------------------------------------------------------------------

## Solusi

### 1. Nonaktifkan 3D sepenuhnya

Ubah XML:

    <video>
      <model type='virtio' heads='1' primary='yes'/>
    </video>

Hapus:

    <acceleration accel3d='yes'/>

------------------------------------------------------------------------

### 2. Pastikan graphics tanpa GL

    <graphics type='spice'>
      <listen type='none'/>
    </graphics>

------------------------------------------------------------------------

### 3. Restart VM

    virsh start kali-linux

------------------------------------------------------------------------

## Jika ingin 3D tetap aktif

Masalah utama: libvirt system mode tidak bisa akses OpenGL environment

### Opsi:

#### Opsi 1: Gunakan session mode

    virsh -c qemu:///session start kali-linux
    virt-viewer --connect qemu:///session kali-linux

#### Opsi 2: Debug libvirt environment

-   cek journalctl
-   cek EGL context
-   cek sandbox/cgroup

------------------------------------------------------------------------

## Kesimpulan Teknis

-   QEMU mendukung virgl
-   Sistem mendukung OpenGL
-   Permission sudah benar

Namun: libvirt (system mode) gagal menjalankan OpenGL

------------------------------------------------------------------------

## Status Akhir

  Komponen            Status
  ------------------- --------
  GPU                 OK
  OpenGL runtime      OK
  QEMU virgl          OK
  Permission          OK
  Libvirt system GL   FAIL
