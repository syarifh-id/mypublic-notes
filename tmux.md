
# Panduan Dasar tmux

`tmux` adalah *terminal multiplexer* yang memungkinkan pengelolaan beberapa sesi terminal dalam satu jendela dan menjaga proses tetap berjalan di latar belakang (sangat berguna untuk SSH).

Hierarki utama dalam `tmux` adalah: **Server** > **Session** > **Window** > **Pane**.

## 1. Tombol Prefix (Kunci Utama)

Hampir semua interaksi di dalam sesi `tmux` membutuhkan tombol Prefix bawaan:
**`Ctrl` + `b**` *(Tekan `Ctrl` dan `b` bersamaan, lepaskan, lalu tekan tombol perintah selanjutnya).*

---

## 2. Manajemen Sesi (Session)

Digunakan untuk mengelompokkan pekerjaan atau proyek.

| Aksi | Perintah Terminal (Luar tmux) | Shortcut (Dalam tmux, setelah `Prefix`) |
| --- | --- | --- |
| **Buat sesi baru** | `tmux new -s <nama_sesi>` | - |
| **Lihat daftar sesi** | `tmux ls` | `s` (Pilih dengan panah, tekan Enter) |
| **Masuk (Attach)** | `tmux attach -t <nama_sesi>` | - |
| **Keluar (Detach)** | - | `d` (Proses tetap berjalan) |
| **Ubah nama sesi** | `tmux rename-session -t <lama> <baru>` | `$` |
| **Tutup sesi** | `tmux kill-session -t <nama_sesi>` | Ketik `exit` di semua terminal |

---


## 3. Manajemen Window Dasar

Secara default, semua perintah di bawah ini harus diawali dengan **Prefix** (biasanya `Ctrl + b`).

| Aksi | Shortcut (`Prefix` + ...) |
| --- | --- |
| **Buat Window baru** | `c` (create) |
| **Ganti nama Window** | `,` (koma) |
| **Tutup Window** | `&` (tanya konfirmasi) atau ketik `exit` |
| **Pindah ke Window berikutnya** | `n` (next) |
| **Pindah ke Window sebelumnya** | `p` (previous) |
| **Pindah berdasarkan angka** | `0-9` |
| **Lihat daftar Window (Interaktif)** | `w` |
| **Cari Window berdasarkan nama** | `f` (find) |

---

### Memahami Struktur Window

Setiap window memiliki indeks (angka) dan nama. Anda bisa melihat status ini di bagian bawah layar (status bar).

* **`0:bash-`**: Menandakan window indeks 0, menjalankan bash, dan merupakan window sebelumnya yang Anda buka.
* **`1:python*`**: Menandakan window indeks 1, dan tanda `*` berarti ini adalah window yang sedang aktif saat ini.

---

### Tips Pro untuk Navigasi Cepat

Jika Anda sering bekerja dengan banyak window, menghafal angka bisa melelahkan. Gunakan trik ini:

* **Navigasi Visual (`Prefix` + `w`):** Ini adalah cara favorit saya. Ini akan membuka menu *overlay* yang memungkinkan Anda melihat pratinjau isi setiap window sebelum memilihnya menggunakan tombol panah.
* **Reorder (Pindah Posisi):** Jika Anda ingin memindahkan window yang aktif ke posisi lain (misalnya dari indeks 4 ke 1), masuk ke command mode (`Prefix` + `:`) lalu ketik:
`swap-window -t 1`

---



## 3. Manajemen Pane (Layar Terbelah)

Digunakan untuk membagi satu layar menjadi beberapa terminal kecil.

| Aksi | Shortcut (Setelah `Prefix`) |
| --- | --- |
| **Belah Vertikal (Kiri/Kanan)** | `%` |
| **Belah Horizontal (Atas/Bawah)** | `"` |
| **Pindah antar Pane** | `Tombol Panah` (Atas/Bawah/Kiri/Kanan) |
| **Zoom Pane (Layar Penuh)** | `z` (Tekan lagi untuk kembali normal) |
| **Tutup Pane aktif** | `x` (Lalu tekan `y` untuk konfirmasi) |

---

## 4. Copy-Mode & Navigasi

Fitur untuk *scroll* riwayat terminal dan *copy-paste* teks menggunakan keyboard.

| Aksi | Shortcut |
| --- | --- |
| **Masuk Copy-Mode** | `Prefix` + `[` |
| **Paste Teks** | `Prefix` + `]` |
| **Keluar / Batal** | `q` |

### Pengaturan Copy-Mode ala Vim

Tambahkan baris berikut ke dalam file `~/.tmux.conf` agar navigasi dan proses seleksi teks menggunakan tombol Vim (`v` untuk *select*, `y` untuk *copy*):

```tmux
# Mengaktifkan tombol vi di copy-mode
setw -g mode-keys vi

# Menggunakan 'v' untuk mulai memblok teks
bind-key -T copy-mode-vi v send-keys -X begin-selection

# Menggunakan 'y' untuk menyalin teks (yank)
bind-key -T copy-mode-vi y send-keys -X copy-selection-and-cancel

```

---

## 5. Memuat Ulang (Reload) Konfigurasi

Perintah untuk menerapkan perubahan pada file `~/.tmux.conf` tanpa perlu mematikan server atau sesi `tmux`.

* **Dari Command Prompt `tmux`:** Tekan `Prefix` + `:`, lalu ketik `source-file ~/.tmux.conf`
* **Dari Terminal Luar:** Ketik `tmux source-file ~/.tmux.conf`

### Shortcut Reload Otomatis

Tambahkan baris ini ke `~/.tmux.conf` agar Anda bisa melakukan *reload* hanya dengan menekan **`Prefix` + `r**`:

```tmux
# Reload konfigurasi tmux dengan Prefix + r
bind r source-file ~/.tmux.conf \; display-message "Konfigurasi dimuat ulang!"

```

---

Apakah Anda ingin saya melengkapi *cheatsheet* di atas dengan daftar pintasan untuk **Manajemen Window** (membuat dan berpindah tab layar penuh)?
