Langkah menghapus semua komponen **VMware Workstation** di **Ubuntu** secara menyeluruh.

1. Hentikan service VMware

```
sudo systemctl stop vmware
sudo systemctl stop vmware-networks
```

2. Jalankan uninstaller resmi

```
sudo vmware-installer -u vmware-workstation
```

Jika perintah tidak ada:

```
sudo /usr/bin/vmware-installer -u vmware-workstation
```

3. Hapus modul kernel VMware

```
sudo rm -rf /lib/modules/$(uname -r)/misc/vmmon.ko
sudo rm -rf /lib/modules/$(uname -r)/misc/vmnet.ko
sudo depmod -a
```

4. Hapus direktori instalasi VMware

```
sudo rm -rf /usr/lib/vmware
sudo rm -rf /etc/vmware
sudo rm -rf /var/lib/vmware
sudo rm -rf /usr/share/vmware
```

5. Hapus service systemd VMware

```
sudo rm -f /etc/systemd/system/vmware.service
sudo rm -f /etc/systemd/system/vmware-networks.service
sudo systemctl daemon-reload
```

6. Hapus konfigurasi user

```
rm -rf ~/.vmware
rm -rf ~/.config/vmware
rm -rf ~/vmware
```

7. Hapus network interface VMware

```
sudo ip link delete vmnet0
sudo ip link delete vmnet1
sudo ip link delete vmnet8
```

8. Hapus file sisa di temporary

```
sudo rm -rf /tmp/vmware*
sudo rm -rf /var/tmp/vmware*
```

9. Verifikasi tidak ada sisa instalasi

```
which vmware
ls /usr/bin | grep vmware
lsmod | grep vm
```

Jika semua berhasil, tidak akan ada lagi binary, modul kernel, maupun konfigurasi VMware yang tersisa di sistem.
