1. Konsep Dasar & Arsitektur
Intel Arc Support: Llama.cpp mendukung GPU Intel Arc (A-series dan iGPU Core Ultra) melalui backend SYCL.
Keunggulan: Menggunakan Intel oneAPI memungkinkan model LLM memanfaatkan unit XMX (Xe Matrix Extensions) untuk akselerasi tensor yang jauh lebih cepat daripada CPU biasa.
VRAM: Intel Arc A770 (16GB) menjadi pilihan populer karena kapasitas VRAM besar untuk menjalankan model AI lokal yang berat.
2. Prasyarat Software
Driver: Membutuhkan driver GPU Intel terbaru.
Toolkit: Wajib menginstal Intel oneAPI Base Toolkit untuk mendapatkan compiler icx (C) dan icpx (C++).
Source Code: Harus melakukan git clone repositori resmi llama.cpp karena akselerasi SYCL memerlukan kompilasi khusus (tidak tersedia di pre-built binary standar).
3. Langkah-Langkah Teknis yang Dilakukan
Aktivasi Environment: Anda telah menginstal oneAPI, namun variabel lingkungan harus diaktifkan setiap kali membuka terminal baru menggunakan perintah:
Linux: source /opt/intel/oneapi/setvars.sh
Windows: Menggunakan oneAPI Command Prompt.
Konfigurasi CMake: Menyiapkan folder build dengan perintah:
cmake -B build -DGGML_SYCL=ON -DCMAKE_C_COMPILER=icx -DCMAKE_CXX_COMPILER=icpx
Proses Kompilasi: Menjalankan perintah build:
cmake --build build --config Release
4. Kendala yang Ditemui & Solusi
Error "Compiler Unknown/Not Found": Terjadi karena terminal belum menjalankan skrip setvars. Solusinya adalah menjalankan skrip tersebut agar icx dan icpx terbaca.
Error "Could not load cache": Terjadi karena mencoba build saat konfigurasi CMake sebelumnya gagal. Solusinya adalah menghapus folder build dan mengulang perintah konfigurasi.
Error Kode add-id.cpp (Bug Sumber): Ditemukan kesalahan penulisan variabel work_group_size (seharusnya max_work_group_size).
Solusi: Melakukan git pull untuk update kode terbaru atau mengedit manual baris 59 pada file tersebut.
5. Cara Penggunaan (Target Akhir)
Setelah build berhasil, model dijalankan dengan perintah:
./bin/llama-cli -m [nama_model].gguf -ngl [jumlah_layer]
Flag -ngl sangat krusial untuk memindahkan beban kerja dari CPU ke GPU Intel Arc.
Apakah Anda ingin saya memberikan perintah pengecekan untuk memastikan GPU Intel Arc Anda sudah terdeteksi dengan benar setelah proses build selesai?




