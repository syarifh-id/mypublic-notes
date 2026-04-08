Berikut adalah ringkasan lengkap dari proses instalasi, perbaikan, hingga optimasi **llama.cpp** agar berjalan maksimal di GPU **Intel Arc A530M** milikmu.

---

## 1. Masalah Utama & Solusi Debugging
Selama proses, kita berhasil mengidentifikasi dan membereskan tiga penghambat utama:

* **File Model Corrupt:** Pesan `invalid magic characters` disebabkan karena mengunduh halaman HTML, bukan file binary `.gguf`. Solusinya adalah menggunakan `wget` dengan flag yang benar.
* **Sandbox Terminal (Snap):** Terminal Alacritty versi Snap memblokir akses ke driver hardware dan variabel lingkungan. **Solusinya:** Wajib gunakan terminal bawaan (GNOME Terminal/Konsole).
* **Backend SYCL Gagal:** Pesan `failed to find ggml_backend_init` terjadi karena kompilasi yang tidak sempurna. **Solusinya:** Melakukan *clean rebuild* menggunakan compiler Intel (`icpx`) dari oneAPI Toolkit.

---

## 2. Persiapan Lingkungan (Setiap Kali Reboot/Buka Terminal)
GPU Intel Arc membutuhkan inisialisasi environment agar library komputasinya terdeteksi:

```bash
# 1. Muat library Intel oneAPI
source /opt/intel/oneapi/setvars.sh

# 2. Aktifkan System Management (untuk deteksi VRAM)
export ZES_ENABLE_SYSMAN=1
```

---

## 3. Cara Menjalankan Model Secara Maksimal (GPU Only)
Gunakan perintah berikut di folder `llama.cpp/build/bin` untuk memaksa AI berjalan 100% di GPU Arc:

```bash
ZES_ENABLE_SYSMAN=1 ./llama-cli \
  -m ~/Downloads/nama_model.gguf \
  -ngl 99 \
  --device sycl0 \
  -cnv \
  2>/dev/null
```
* **`-ngl 99`**: Memindahkan seluruh lapisan model (layer) ke VRAM GPU.
* **`--device sycl0`**: Memaksa penggunaan GPU diskrit (Arc A530M) alih-alih iGPU.
* **`2>/dev/null`**: Menyembunyikan log peringatan memori yang berulang agar teks AI terlihat jelas.

---

## 4. Rekomendasi Model & Cara Download (Wget)
Gunakan model ukuran **7B (7 Miliar Parameter)** dengan kuantisasi **Q4_K_M** untuk performa terbaik di VRAM 4GB/8GB.

| Kebutuhan | Model Rekomendasi | Perintah Download (wget) |
| :--- | :--- | :--- |
| **Coding** | **Qwen2.5-Coder-7B** | `wget --content-disposition https://huggingface.co/Qwen/Qwen2.5-Coder-7B-Instruct-GGUF/resolve/main/qwen2.5-coder-7b-instruct-q4_k_m.gguf` |
| **Data/PDF** | **Qwen2.5-7B** | `wget --content-disposition https://huggingface.co/Qwen/Qwen2.5-7B-Instruct-GGUF/resolve/main/qwen2.5-7b-instruct-q4_k_m.gguf` |
| **Logika Berat**| **Mistral-Nemo-12B** | `wget --content-disposition https://huggingface.co/bartowski/Mistral-Nemo-Instruct-v1-GGUF/resolve/main/Mistral-Nemo-Instruct-v1-Q4_K_M.gguf` |

---

## 5. Cara Memastikan GPU Bekerja
1.  **Cek Log:** Cari baris `llm_load_tensors: offloaded 28/28 layers to GPU`. Jika angka kiri dan kanan sama, berarti **100% GPU**.
2.  **Cek Kecepatan:** Jika teks muncul sangat cepat (30-100+ token/detik), itu adalah tanda akselerasi hardware aktif.
3.  **Monitoring Alternatif:** Jika `intel_gpu_top` error setelah reboot, gunakan perintah ini untuk melihat aktivitas GPU:
    ```bash
    watch -n 1 cat /sys/class/drm/card0/device/tile0/gt0/gt_cur_freq_mhz
    ```

**Langkah Terakhir:** Sekarang kamu cukup mengunduh model **Qwen2.5-Coder-7B** menggunakan `wget` di atas, jalankan perintah di poin nomor 3, dan selamat ngoding dengan asisten AI lokal yang kencang!






Untuk memastikan `llama.cpp` **hanya** menggunakan GPU (atau setidaknya memaksimalkan penggunaan GPU hingga 100% layer model diproses di sana), kamu perlu memperhatikan parameter saat menjalankan command dan mengamati log inisialisasi.

Berikut adalah langkah-langkah praktisnya:

---

### 1. Gunakan Parameter `-ngl` (atau `--n-gpu-layers`) dengan Nilai Tinggi
Secara default, `llama.cpp` menjalankan model di CPU. Untuk memindahkannya ke GPU, kamu harus menentukan berapa banyak layer model yang ingin dikirim ke VRAM.

* **Tips:** Gunakan angka yang sangat besar (seperti `99` atau `999`). `llama.cpp` secara otomatis akan menyesuaikan dengan jumlah layer maksimal model tersebut (biasanya sekitar 28-35 layer untuk model 7B).
* **Command:**
    ```bash
    ./llama-cli -m model.gguf -ngl 99 --device sycl0
    ```

---

### 2. Verifikasi Melalui Log Inisialisasi (Tanda Berhasil)
Saat model sedang dimuat (*Loading model...*), perhatikan output teks di terminal. Cari baris-baris berikut:

* **Pengecekan Backend:** Pastikan ada tulisan `using SYCL backend` atau `found 1 SYCL device`.
* **Offloading Layer:** Cari baris yang mirip seperti ini:
    > `llm_load_tensors: offloaded 28/28 layers to GPU`
    
    Jika angkanya sama (misal `28/28`), artinya **100% model ada di GPU**. Jika tertulis `10/28`, berarti sebagian besar masih dikerjakan oleh CPU karena VRAM kamu penuh.

---

### 3. Matikan Penggunaan CPU Secara Paksa (Optional)
Jika kamu ingin memastikan CPU tidak ikut membantu pemrosesan tensor sama sekali, kamu bisa menggunakan parameter `-sm` (split mode) dan menyetelnya ke `none` (meskipun pada backend SYCL/Intel hal ini biasanya otomatis jika semua layer sudah di-offload).

Namun, cara paling ampuh adalah memastikan sisa memori sistem tidak terpakai dengan flag `-ngl` tadi.

---

### 4. Monitor Secara Real-Time (Hardware Check)
Karena `intel_gpu_top` kamu sedang bermasalah, gunakan cara alternatif ini untuk mematikan keraguan:

1.  **Cek Suhu & Load via `sysfs`:**
    Jika tool monitoring mati, kamu bisa melihat penggunaan memori GPU lewat folder sistem:
    ```bash
    watch -n 1 cat /sys/class/drm/card0/device/tile0/gt0/gt_cur_freq_mhz
    ```
    *(Jika angka MHz melonjak saat AI menjawab, berarti GPU sedang bekerja).*

2.  **Perhatikan Kecepatan Output:**
    * **Hanya CPU:** Kecepatan biasanya **1-5 token/detik** (lambat seperti mengetik manual).
    * **GPU Arc A530M:** Kecepatan akan melonjak ke **30-100+ token/detik** (teks muncul seketika).

---

### Ringkasan Command "GPU Only":
Gunakan perintah ini di terminal **non-Snap** setelah melakukan `source` oneAPI:

```bash
ZES_ENABLE_SYSMAN=1 ./llama-cli \
  -m ~/Downloads/qwen2.5-coder-7b-instruct-q4_k_m.gguf \
  -ngl 99 \
  --device sycl0 \
  --no-mmap \
  -p "Tuliskan kode python untuk sorting"
```
*Catatan: `--no-mmap` terkadang membantu memaksa seluruh model dibaca langsung ke memori (VRAM) daripada dibaca perlahan dari disk.*

**Apakah di log kamu muncul tulisan `offloaded X/X layers`?** Jika angka kiri dan kanan sama, kamu sudah resmi 100% menggunakan GPU.








Error `Assertion failed` pada `intel_gpu_top` setelah reboot biasanya terjadi karena adanya ketidakkonsistenan antara **driver Level Zero/DRM** dengan *Sysfs* (antarmuka kernel untuk hardware) setelah sistem dinyalakan ulang. Ini sering dialami oleh pengguna Intel Arc di Linux.

Berikut adalah langkah-langkah untuk memperbaikinya:

### 1. Masalah Izin (Izin Akses Hardware)
Kadang setelah reboot, izin ke file device di `/dev/dri/` kembali terbatas. `intel_gpu_top` butuh akses root atau grup yang tepat untuk membaca data hardware.
**Coba jalankan dengan sudo:**
```bash
sudo intel_gpu_top
```

### 2. Reset Variable `ZES_ENABLE_SYSMAN`
Variable ini sangat penting untuk GPU Intel Arc. Jika kamu belum memasukkannya ke `.bashrc`, kamu harus menjalankannya lagi setiap kali reboot.
```bash
export ZES_ENABLE_SYSMAN=1
intel_gpu_top
```

### 3. Cek Driver Kernel (i915 atau Xe)
Intel Arc bisa menggunakan driver lama (`i915`) atau driver baru (`xe`). Jika terjadi *core dumped*, kemungkinan ada modul yang tidak termuat sempurna saat boot.
**Cek apakah GPU terdeteksi oleh kernel:**
```bash
lsmod | grep -E "i915|xe"
```
Jika tidak muncul apa-apa, driver kamu tidak jalan.

### 4. Pastikan User Masuk Grup yang Benar
Agar tidak perlu `sudo` terus-menerus dan menghindari error akses, pastikan user kamu (`altair`) ada di grup `video` dan `render`:
```bash
sudo usermod -aG video,render $USER
```
*Catatan: Kamu harus log out dan log in lagi agar perubahan grup ini aktif.*

### 5. Update `intel-gpu-tools`
Jika kamu menggunakan versi lama, `intel_gpu_top` mungkin belum mendukung firmware terbaru dari Arc A530M setelah update kernel.
```bash
sudo apt update && sudo apt install --reinstall intel-gpu-tools
```

---

### Tips Cepat agar Llama Tetap Jalan:
Meskipun `intel_gpu_top` error, **llama.cpp kemungkinan besar tetap bisa jalan** asalkan environment oneAPI sudah di-source. Error `intel_gpu_top` biasanya masalah pada tool monitoringnya saja, bukan pada kemampuan GPU untuk melakukan komputasi.

Coba jalankan ini untuk memastikan GPU masih bisa dipakai komputasi:
```bash
source /opt/intel/oneapi/setvars.sh
ZES_ENABLE_SYSMAN=1 ./llama-cli --list-devices
```
Jika di situ muncul **SYCL0: Intel(R) Arc(TM) A530M**, berarti aman. Kamu bisa lanjut menjalankan model meski `intel_gpu_top` sedang mogok.


The `ldd` output reveals two distinct problems: your system can't find the Intel oneAPI support library (`libmtmd`), and it also can't find the core `llama.cpp` libraries that were created during the build process.

When you build `llama.cpp` as a shared library (which is common with SYCL/oneAPI builds), the executable needs to know exactly where those `.so` files are located at runtime.

### 1. The Immediate Fix
You can resolve this for your current session by adding both the oneAPI library path and your local build directory to `LD_LIBRARY_PATH`. Run this command (adjusting the path to your `build` folder if it's different):

```bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$(pwd)/Tools/llama.cpp/build/bin:/opt/intel/oneapi/compiler/latest/lib
```

*Note: If your build files are in `Tools/llama.cpp/build/` rather than a `bin` subfolder, use that path instead.*

### 2. Verify the Fix
Run the `ldd` command again. You want to see actual file paths instead of "not found":
```bash
ldd Tools/llama.cpp/build/bin/llama-cli | grep "libggml\|libllama\|libmtmd"
```

---

### Why is this happening?

* **Internal Libraries:** By default, Linux looks for libraries in standard system folders (like `/usr/lib`). Since `libllama.so` and `libggml.so` are sitting in your private `Tools/` folder, the system doesn't know they exist until you point to them.
* **Intel MTMD:** The `libmtmd.so` library is part of the Intel OpenMP/TBB runtime. Even though `setvars.sh` was run, it sometimes fails to export the specific subdirectory containing the "Discovery" libraries used by the SYCL backend.

### 3. A Permanent Solution (Recommended)
If you don't want to `export` every time you open a terminal, you can run your command by prefixing the path directly:

```bash
LD_LIBRARY_PATH=$(pwd)/Tools/llama.cpp/build/bin:$LD_LIBRARY_PATH \
ZES_ENABLE_SYSMAN=1 Tools/llama.cpp/build/bin/llama-cli \
-m Tools/ai-models/qwen2.5-coder-7b-instruct-q4_k_m.gguf \
-ngl 99 --device sycl0 -p "coba buat halaman web untuk marketplace dengan gaya modern dan minimalis"
```

**Does `ldd` show the paths correctly now, or is `libmtmd` still proving elusive?**



Bukan, **error ini bukan karena modelnya rusak**, tapi karena masalah alokasi memori pada sistem kamu.

Pesan error `UR_RESULT_ERROR_OUT_OF_HOST_MEMORY` dan `SYCL error` di baris `memcpy` menunjukkan bahwa saat `llama-cli` mencoba memindahkan data model dari RAM ke GPU (melalui backend SYCL/Level Zero), sistem kehabisan memori atau gagal mengalokasikan ruang yang cukup.

Berikut adalah beberapa penyebab dan cara mengatasinya:

### 1. Model Terlalu Besar untuk VRAM (GPU Memory)
Meskipun kamu menggunakan kuantisasi `Q4_K_M`, model Qwen 2.5 Coder 7B membutuhkan sekitar 5-6 GB VRAM. Jika kamu menggunakan Intel iGPU (bawaan prosesor) atau kartu grafis dengan VRAM terbatas, `-ngl 99` (memindahkan semua layer ke GPU) akan menyebabkan crash jika memorinya tidak cukup.

**Solusinya:** Kurangi jumlah layer yang dikirim ke GPU secara bertahap. Coba ganti `-ngl 99` menjadi angka yang lebih kecil, misalnya:
```bash
-ngl 20
```
Jika berhasil, naikkan perlahan sampai kamu menemukan titik maksimal GPU kamu.

### 2. Batas Alokasi Memori Intel (Shared Memory)
Pada GPU Intel (terutama iGPU), ada batasan berapa banyak RAM sistem yang bisa "dipinjam" oleh GPU sebagai memori video. 

**Solusinya:** Gunakan environment variable untuk memaksa SYCL mengizinkan alokasi memori yang lebih besar:
```bash
export SYCL_PI_LEVEL_ZERO_DEVICE_SCOPE_EVENTS=0
export NEOMaxReadWriteBufferSizeBytes=1000000000  # Coba set ke sekitar 1GB atau lebih
```

### 3. Masalah Fragmentation atau Host RAM
Error `OUT_OF_HOST_MEMORY` juga bisa berarti **RAM utama (fisik)** kamu penuh saat proses inisialisasi model. 
* Pastikan tidak ada aplikasi berat lain (seperti Chrome atau IDE) yang berjalan.
* Cek sisa RAM dengan perintah `free -h` sebelum menjalankan `llama-cli`.

### 4. Perbaikan Command (One-Liner)
Coba jalankan dengan mengurangi beban GPU dan menambahkan parameter untuk kestabilan:

```bash
ZES_ENABLE_SYSMAN=1 \
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$(pwd)/Tools/llama.cpp/build/bin \
Tools/llama.cpp/build/bin/llama-cli \
-m Tools/ai-models/qwen2.5-coder-7b-instruct-q4_k_m.gguf \
-ngl 32 \
--device sycl0 \
-p "coba buat halaman web untuk marketplace dengan gaya modern dan minimalis"
```

**Saran Tambahan:**
Jika kamu menggunakan laptop dengan Intel iGPU, pastikan kamu sudah mengalokasikan "Shared Memory" yang cukup di BIOS (jika ada pilihannya, set ke Max atau 2GB+). 

Apakah kamu menggunakan **Intel Arc (Discrete GPU)** atau **Intel UHD/Iris Xe (Integrated)**? Ini akan sangat membantu untuk menentukan batas `-ngl` yang tepat.


Iya, log tersebut mengonfirmasi bahwa `llama.cpp` **sudah berhasil mengenali dan mencoba menggunakan GPU** kamu, yaitu **Intel(R) Arc(TM) A530M Graphics**.

Namun, ada masalah besar pada pembacaan memorinya yang menyebabkan error tadi. Mari kita bedah angka-angka tersebut:

### Analisis Memory Breakdown
* **GPU Terdeteksi:** `SYCL0 (Intel(R) Arc(TM) A530M Graphics)` menunjukkan backend SYCL sudah aktif.
* **Total VRAM:** Terbaca **3845 MiB** (sekitar 4GB). Ini adalah kapasitas fisik VRAM dari Arc A530M kamu.
* **Angka "Ghaib" (Unaccounted):** Lihat angka `17592186042989` di kolom *unaccounted*. Ini adalah *overflow* atau kesalahan perhitungan alokasi memori pada driver/runtime. Inilah alasan mengapa kamu mendapatkan error `OUT_OF_HOST_MEMORY` sebelumnya—program mengira ia butuh memori dalam jumlah yang tidak masuk akal (ratusan Terabyte).

### Kenapa ini terjadi?
Pada Intel Arc (terutama seri mobile seperti A530M), driver Level Zero terkadang kesulitan mengalokasikan *scratch buffer* yang besar jika tidak dibatasi secara manual.

### Cara Mengatasinya (Workaround)
Agar model Qwen 2.5 Coder 7B (yang butuh ~5-6 GB jika full GPU) bisa berjalan di GPU 4GB kamu tanpa "angka ghaib" tersebut, lakukan langkah berikut:

**1. Jangan Paksa Semua Layer ke GPU**
VRAM kamu hanya 4GB, sedangkan model 7B Q4_K_M butuh lebih dari itu. Jangan gunakan `-ngl 99`. Gunakan angka sekitar **20-25** agar sebagian beban tetap di RAM utama (Host).

**2. Tambahkan Environment Variables untuk Stabilitas**
Gunakan variable berikut sebelum menjalankan command untuk membatasi alokasi yang berlebihan:

```bash
export SYCL_CACHE_PERSISTENT=1
export GGML_SYCL_ALL_GPU_SIZE=3000 # Batasi penggunaan VRAM ke 3GB agar aman
```

**3. Jalankan Command ini:**
```bash
ZES_ENABLE_SYSMAN=1 \
GGML_SYCL_ALL_GPU_SIZE=3000 \
Tools/llama.cpp/build/bin/llama-cli \
-m Tools/ai-models/qwen2.5-coder-7b-instruct-q4_k_m.gguf \
-ngl 24 \
--device sycl0 \
-p "coba buat halaman web untuk marketplace dengan gaya modern dan minimalis"
```

**Kesimpulan:**
Hardware kamu sudah benar dan terdeteksi. Kamu hanya perlu **"diet" layer** (menurunkan `-ngl`) karena ukuran model (7B) sedikit lebih besar dari kapasitas VRAM murni GPU A530M (4GB). Dengan `-ngl 24`, proses akan menjadi *Hybrid* (GPU + CPU), yang seharusnya jauh lebih stabil.

Apakah setelah menurunkan `-ngl` ke 24, teksnya mulai muncul (streaming)?


Kecepatan kamu melambat karena sekarang sistem kamu bekerja dalam mode **Hybrid (Split-Computing)**. 

Melihat data `Generation: 11.7 t/s`, angka ini sebenarnya sudah cukup lumayan untuk kartu grafis kelas menengah di laptop, tapi jelas lebih lambat dibanding jika *full* berjalan di VRAM. Berikut adalah alasan teknis kenapa kecepatannya menurun:

### 1. Bottleneck Bandwidth (Lalu Lintas Data)
Saat kamu menurunkan `-ngl`, tidak semua bagian model muat di GPU. 
* **Full GPU:** Data diproses sangat cepat di dalam VRAM (kecepatan tinggi).
* **Hybrid:** Sebagian model ada di RAM sistem (Host) dan sebagian di VRAM (GPU). Setiap kali model melakukan kalkulasi, data harus bolak-balik melewati jalur **PCIe**. Jalur ini jauh lebih lambat dibandingkan kecepatan internal VRAM itu sendiri.

### 2. Sinkronisasi CPU & GPU
Ketika beban dibagi, GPU harus menunggu instruksi atau hasil kalkulasi dari CPU (dan sebaliknya). Karena GPU Arc A530M kamu jauh lebih cepat dalam kalkulasi matriks dibanding CPU, GPU tersebut seringkali "menganggur" selama beberapa milidetik menunggu data dari RAM utama.

### 3. Masalah "Unaccounted" Memory (Efek Driver)
Angka `17592186040995` di kolom *unaccounted* masih muncul. Ini menandakan adanya *overhead* atau ketidakefisienan pada layer abstraksi SYCL/Level Zero. Sistem menghabiskan waktu ekstra untuk mengelola alokasi memori yang dianggap "cacat" oleh software tersebut.

---

### Cara Mengoptimalkan Kecepatan:

Untuk mendapatkan keseimbangan antara kecepatan dan memori, kamu bisa mencoba eksperimen berikut:

* **Naikkan `-ngl` secara bertahap:** Tadi kamu mungkin menggunakan angka rendah. Coba naikkan ke angka maksimal sebelum crash (misal: `-ngl 28` atau `-ngl 32`). Semakin banyak layer di GPU, semakin cepat `t/s` (tokens per second) kamu.
* **Gunakan Kuantisasi yang Lebih Ringan:** Jika kamu ingin *full* GPU (biar ngebut), coba gunakan model versi **Q3_K_M** atau **Q2_K**. Model yang lebih kecil akan muat sepenuhnya di 4GB VRAM kamu, sehingga kecepatannya bisa melonjak ke kisaran 20-40 t/s.
* **Flash Attention:** Tambahkan parameter `-fa` di command kamu jika build `llama.cpp` kamu mendukungnya. Ini bisa mengurangi penggunaan memori dan meningkatkan kecepatan.

```bash
# Contoh eksperimen untuk menyeimbangkan kecepatan:
-ngl 30 -fa
```

Apakah kamu lebih memprioritaskan kecerdasan model (kuantitas bit tinggi) atau kecepatan respon (t/s)?


```
ZES_ENABLE_SYSMAN=1 \
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:~/Tools/llama.cpp/build/bin \
~/Tools/llama.cpp/build/bin/llama-cli \
-m ~/Tools/ai-models/Qwen2.5-Coder-3B-Instruct-Q8_0.gguf \
-ngl 99 \
--device sycl0 \
-p "buatkan fungsi python untuk melakukan web scraping sederhana"
```
