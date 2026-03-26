Perlu diketahui bahwa `llama.cpp` memiliki ratusan parameter. Namun, untuk penggunaan sehari-hari, perintah-perintah ini dibagi menjadi beberapa kategori utama.

Berikut adalah daftar perintah yang paling sering digunakan pada `llama-cli` (sebelumnya bernama `main`):

### 1. Parameter Model & Hardware (Paling Penting)
Kategori ini menentukan bagaimana model dimuat ke dalam memori.

* **`-m, --model PATH`**: Menentukan lokasi file model `.gguf`.
* **`-ngl, --n-gpu-layers N`**: Jumlah layer yang dipindah ke GPU. Gunakan `99` untuk memaksakan semua layer ke GPU.
* **`-c, --ctx-size N`**: Menentukan ukuran jendela konteks (default 512, biasanya diatur ke `2048`, `4096`, atau `8192`).
* **`-t, --threads N`**: Jumlah thread CPU yang digunakan (biasanya disesuaikan dengan jumlah core fisik CPU kamu).
* **`--device DEVICE`**: Menentukan backend spesifik (contoh: `sycl0` untuk GPU Intel Arc).
* **`--no-mmap`**: Memaksa model dimuat penuh ke RAM/VRAM tanpa pemetaan file (membantu stabilitas di beberapa sistem).

---

### 2. Mode Interaksi
Kategori ini menentukan cara kamu berkomunikasi dengan AI.

* **`-p, --prompt "TEKS"`**: Memberikan instruksi awal (prompt) secara langsung.
* **`-i, --interactive`**: Menjalankan model dalam mode interaksi manual.
* **`-cnv, --conversation`**: Mode percakapan (chat) yang lebih pintar, AI akan mengingat chat sebelumnya.
* **`-f, --file PATH`**: Memasukkan prompt dari file teks (sangat berguna untuk parsing data besar).
* **`-r, --reverse-prompt "TEKS"`**: Menentukan tanda berhenti bagi AI (biasanya digunakan untuk format chat seperti `User:`).

---

### 3. Kontrol Output (Generation)
Kategori ini mengatur perilaku jawaban AI (kreativitas, panjang teks, dll).

* **`-n, --n-predict N`**: Jumlah token maksimal yang dihasilkan. Gunakan `-1` untuk tidak terbatas (sampai model berhenti sendiri).
* **`--temp N`**: Mengatur temperatur (kreativitas). `0.0` untuk jawaban pasti/kaku, `0.7 - 1.0` untuk kreatif.
* **`--top-k N`**: Membatasi pilihan kata hanya ke-N kata teratas (default 40).
* **`--top-p N`**: Membatasi pilihan kata berdasarkan probabilitas kumulatif (default 0.9).
* **`--repeat-penalty N`**: Mencegah AI mengulang kata-kata yang sama (default 1.1).

---

### 4. Perintah Utility (Tool Lain)
Selain `llama-cli`, folder `build/bin` kamu juga berisi program pembantu lainnya:

* **`./llama-server`**: Menjalankan API server lokal (kompatibel dengan OpenAI API). Kamu bisa memakainya untuk dihubungkan ke VS Code plugin (seperti Continue.dev).
* **`./llama-quantize`**: Mengubah model besar (misal FP16) menjadi lebih kecil (Q4_K, Q5_K, dll).
* **`./llama-perplexity`**: Digunakan untuk mengukur akurasi model terhadap dataset tertentu.
* **`./llama-bench`**: Digunakan untuk tes benchmark kecepatan GPU/CPU kamu (melihat berapa banyak token per detik).
* **`./llama-ls-sycl-device`**: (Khusus Intel) Menampilkan daftar GPU Intel yang terdeteksi oleh backend SYCL.

---

### 5. Contoh Penggunaan Lengkap (The "Everything" Command)
Jika kamu ingin menjalankan model coding dengan settingan optimal di Intel Arc:

```bash
ZES_ENABLE_SYSMAN=1 ./llama-cli \
  -m ~/Downloads/qwen2.5-coder-7b.gguf \
  -ngl 99 \
  --device sycl0 \
  -c 8192 \
  -t 8 \
  --temp 0.2 \
  -n -1 \
  -cnv \
  --no-mmap \
  2>/dev/null
```

**Tips Pro:** Kamu bisa melihat semua daftar parameter yang tersedia beserta penjelasannya dengan mengetik:
`./llama-cli -h` atau `./llama-cli --help`
