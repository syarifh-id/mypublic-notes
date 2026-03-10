# LXC & LXD Discussion

## Q: apakah kamu tau lxc

A: Ya.

LXC adalah singkatan dari Linux Containers, yaitu teknologi virtualisasi
berbasis container yang berjalan langsung di kernel Linux. LXC
memungkinkan kita menjalankan beberapa sistem Linux terisolasi
(container) di dalam satu host tanpa menggunakan hypervisor seperti pada
virtual machine.

Karakteristik: 1. Berbasis kernel Linux (cgroups dan namespaces). 2.
Lebih ringan dibanding virtual machine. 3. Setiap container memiliki
filesystem, network stack, dan proses terisolasi. 4. Cocok untuk
development, lab, maupun production tertentu.

------------------------------------------------------------------------

## Q: cara install di ubuntu

A:

1.  Update repository sudo apt update sudo apt upgrade -y

2.  Install LXC sudo apt install lxc -y

3.  Cek versi lxc-checkconfig lxc-ls --version

4.  Buat container sudo lxc-create -n mycontainer -t download

5.  Jalankan container sudo lxc-start -n mycontainer sudo lxc-attach -n
    mycontainer

Alternatif LXD: sudo snap install lxd sudo lxd init lxc launch
ubuntu:22.04 mycontainer

------------------------------------------------------------------------

## Q: apa bedanya dengan lxd

A:

LXC: - Tool low-level container. - Konfigurasi manual. - Fokus runtime
container.

LXD: - Manajemen modern di atas LXC. - API dan CLI lebih mudah. -
Mendukung snapshot, storage backend, clustering. - Bisa container dan
virtual machine.

------------------------------------------------------------------------

## Q: oke sekarang kita buat container pertama

A:

1.  Install dan init LXD sudo snap install lxd sudo lxd init

2.  Tambah user ke group sudo usermod -aG lxd \$USER newgrp lxd

3.  Buat container lxc launch ubuntu:22.04 mycontainer

4.  Masuk container lxc exec mycontainer -- bash

5.  Stop / Start lxc stop mycontainer lxc start mycontainer

------------------------------------------------------------------------

## Q: lanjut

A:

Cek IP: lxc list lxc info mycontainer

Static IP: lxc config device set mycontainer eth0 ipv4.address 10.0.0.50
lxc restart mycontainer

Expose port: lxc config device add mycontainer myport proxy
listen=tcp:0.0.0.0:8080 connect=tcp:127.0.0.1:80

Install nginx: apt install nginx -y

------------------------------------------------------------------------

## Q: list distro lxc

A:

List semua image: lxc image list images:

Filter distro: lxc image list images: ubuntu lxc image list images:
debian lxc image list images: alpine lxc image list images: archlinux
lxc image list images: rockylinux lxc image list images: centos

------------------------------------------------------------------------

## Q: apakah lxc juga ada mode virtual machine

A:

LXC: hanya container. LXD: mendukung container dan virtual machine.

Buat VM: lxc launch ubuntu:22.04 myvm --vm

Cek tipe: lxc list

------------------------------------------------------------------------

## Q: jika menggunakan type Virtual-machine bagaimana agar tampil GUI

A:

1.  Buat VM lxc launch ubuntu:22.04 myvm --vm -c limits.memory=4GiB -c
    limits.cpu=2

2.  Install desktop apt update apt install xfce4 xfce4-goodies -y

atau apt install ubuntu-desktop -y

3.  Akses GUI lxc console myvm --type=vga

Alternatif RDP: apt install xrdp -y lxc config device add myvm rdp proxy
listen=tcp:0.0.0.0:3389 connect=tcp:127.0.0.1:3389
