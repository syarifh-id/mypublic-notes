# Ringkasan Percakapan: Konfigurasi Nginx

## 1. Konfigurasi Awal Nginx (Static + Backend)

Konfigurasi awal berfungsi untuk: - Melayani **file statis** dari
direktori server. - Meneruskan request yang tidak ditemukan ke **backend
application**.

Komponen utama: - `listen 80` → server menerima HTTP pada port 80 -
`server_name` → domain yang dilayani - `root` → direktori file statis -
`try_files` → mencari file statis terlebih dahulu - `@backend` →
fallback ke aplikasi backend - `proxy_pass` → meneruskan request ke
server aplikasi

Alur request: 1. Client mengakses domain. 2. Nginx mencari file di
direktori root. 3. Jika ada → dikirim sebagai static file. 4. Jika tidak
ada → diteruskan ke backend (`localhost:8080`).

## 2. Konfigurasi Sederhana untuk Static File

Versi konfigurasi yang hanya melayani file statis.

Fitur utama: - Root directory untuk file website. - Default file
`index.html`. - File yang tidak ada akan menghasilkan `404`. - Asset
statis dapat menggunakan cache browser.

Contoh asset yang biasa dicache: - CSS - JavaScript - Image - Video

## 3. Menggunakan HTTPS (Port 443)

Server dapat mendengarkan pada port **443** dengan menambahkan
konfigurasi SSL.

Komponen penting: - `listen 443 ssl` - `ssl_certificate` -
`ssl_certificate_key`

Tanpa sertifikat SSL, Nginx tidak dapat berjalan pada port HTTPS.

Biasanya sertifikat disimpan di:

    /etc/nginx/ssl/

## 4. Konfigurasi Nginx Lengkap untuk Static Website

Fitur konfigurasi lengkap: - Redirect HTTP ke HTTPS - SSL/TLS aktif -
Security headers - Cache untuk asset statis - Logging access dan error -
Proteksi file tersembunyi

Arsitektur umum:

Client → Nginx → Static Files

## 5. Konfigurasi Nginx sebagai Reverse Proxy

Nginx juga dapat digunakan sebagai **reverse proxy** untuk aplikasi
backend.

Contoh backend: - Node.js - Python (Flask, FastAPI, Django) - Java
(Spring Boot)

Fungsi utama reverse proxy: - Terminasi SSL - Forward request ke
backend - Menyimpan IP asli client - Mendukung WebSocket

Arsitektur umum:

Client → Nginx → Backend Application

## 6. Header Proxy Penting

Header yang sering digunakan:

-   `Host`
-   `X-Real-IP`
-   `X-Forwarded-For`
-   `X-Forwarded-Proto`

Tujuannya agar backend mengetahui: - domain asli - alamat IP client -
protokol HTTP/HTTPS

## 7. Optimasi Static File

Beberapa optimasi yang digunakan:

-   `expires 30d` → cache browser
-   `sendfile on` → transfer file lebih cepat
-   `tcp_nopush on` → optimasi pengiriman TCP
-   `access_log off` → mengurangi log untuk asset statis

## 8. Pola Arsitektur yang Dibahas

### Static Website

Client → Nginx → File System

### Reverse Proxy Web App

Client → Nginx → Backend Server

### Hybrid

Client → Nginx → Static File / Backend Application
