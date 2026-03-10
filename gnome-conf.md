Untuk menghilangkan menu **Recent** pada file manager di Ubuntu (biasanya Nautilus), ada beberapa pendekatan tergantung kebutuhan.

## 1. Menonaktifkan Penyimpanan Riwayat (Disarankan)

Ini akan menghentikan pencatatan file yang baru dibuka.

Jalankan:

```bash
gsettings set org.gnome.desktop.privacy remember-recent-files false
```

Untuk mengaktifkan kembali:

```bash
gsettings set org.gnome.desktop.privacy remember-recent-files true
```

---

## 2. Menghapus Daftar Recent yang Sudah Ada

Hapus file database recent:

```bash
rm ~/.local/share/recently-used.xbel
```

Kemudian restart Nautilus:

```bash
nautilus -q
```

---

## 3. Menghilangkan Menu “Recent” dari Sidebar (Hard Disable via CSS Override)

GNOME tidak menyediakan opsi resmi untuk menyembunyikan item sidebar secara selektif. Jika ingin benar-benar menghilangkan tampilannya, perlu modifikasi tema atau patch Nautilus, yang tidak direkomendasikan karena bisa rusak saat update.

Alternatif yang lebih aman adalah pendekatan nomor 1 (disable tracking), sehingga menu tetap ada tapi kosong.

---

Jika yang Anda maksud adalah file manager lain (misalnya Nemo atau Thunar), mekanismenya berbeda.


Untuk membuat shortcut aplikasi **.AppImage** di GNOME (misalnya aplikasi video editor yang dijalankan dengan `./nama-aplikasi.AppImage`), lakukan langkah berikut.

1. Pindahkan file AppImage ke lokasi tetap (opsional tapi disarankan)

Contoh:

```
mkdir -p ~/Applications
mv nama-aplikasi.AppImage ~/Applications/
chmod +x ~/Applications/nama-aplikasi.AppImage
```

Pastikan sudah executable:

```
chmod +x ~/Applications/nama-aplikasi.AppImage
```

2. Buat file shortcut (.desktop)

Buat file:

```
nano ~/.local/share/applications/nama-aplikasi.desktop
```

Isi dengan format berikut:

```
[Desktop Entry]
Name=Nama Aplikasi
Comment=Video Editing Application
Exec=/home/username/Applications/nama-aplikasi.AppImage
Icon=/home/username/Applications/icon.png
Terminal=false
Type=Application
Categories=AudioVideo;Video;Editing;
StartupNotify=true
```

Penjelasan penting:

* Ganti `/home/username` dengan path user Anda.
* `Exec` harus path absolut, tidak boleh menggunakan `./`
* `Icon` bisa diisi path file .png/.svg jika tersedia. Jika tidak ada, baris ini bisa dihapus.

3. Berikan permission pada file .desktop

```
chmod +x ~/.local/share/applications/nama-aplikasi.desktop
```

4. Reload GNOME (jika belum muncul)

Logout-login ulang atau jalankan:

```
update-desktop-database ~/.local/share/applications/
```

Setelah itu aplikasi akan muncul di menu GNOME dan bisa dipin ke dock.
