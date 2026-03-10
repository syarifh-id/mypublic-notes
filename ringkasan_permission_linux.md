# Ringkasan Percakapan: Permission File di Linux

## 1. Memperbaiki File yang Menjadi Executable

Di Linux, file yang menjadi executable biasanya hanya karena perubahan
permission, bukan karena isi file berubah.

Langkah pengecekan:

    ls -l nama_file

Jika terdapat permission `x`, maka file dapat dieksekusi.

Menghilangkan executable:

    chmod -x nama_file

Atau mengatur permission standar file biasa:

    chmod 644 nama_file

------------------------------------------------------------------------

## 2. Mengubah Permission Banyak File Sekaligus

### Beberapa file spesifik

    chmod 644 file1 file2 file3

### Semua file dalam folder

    chmod 644 *

### Berdasarkan ekstensi

Contoh semua file `.txt`:

    chmod 644 *.txt

### Menghapus executable dari banyak file

    chmod -x *

------------------------------------------------------------------------

## 3. Perbedaan Permission File dan Folder

Permission file dan folder tidak boleh disamakan.

Prinsip umum: - File biasa: `644` → `rw-r--r--` - Folder: `755` →
`rwxr-xr-x`

Folder membutuhkan permission `x` agar bisa diakses (`cd`, `ls`).

------------------------------------------------------------------------

## 4. Mengatur Permission File dan Folder Secara Terpisah

Gunakan `find` agar file dan folder diperlakukan berbeda.

### Mengatur permission file

    find nama_folder -type f -exec chmod 644 {} \;

### Mengatur permission folder

    find nama_folder -type d -exec chmod 755 {} \;

------------------------------------------------------------------------

## 5. Memperbaiki Jika Salah Mengatur Permission Rekursif

Jika sebelumnya menjalankan:

    chmod -R 644 nama_folder

Folder akan kehilangan permission `x` dan tidak bisa diakses.

Perbaikan:

    find nama_folder -type d -exec chmod +x {} \;

Atau set ulang secara benar:

    find nama_folder -type d -exec chmod 755 {} \;
    find nama_folder -type f -exec chmod 644 {} \;

------------------------------------------------------------------------

## 6. Menghapus Executable Hanya dari File

Jika banyak file menjadi executable tetapi folder tidak ingin diubah:

    find nama_folder -type f -exec chmod -x {} \;
