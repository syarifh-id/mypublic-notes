
# Setup Remote Ubuntu dengan TigerVNC + XFCE + noVNC

Ringkasan konfigurasi agar Ubuntu dapat diremote melalui browser menggunakan noVNC.

Tanggal: 2026-03-14T18:01:04.817570 UTC

---

# 1. Instalasi

sudo apt update

sudo apt install tigervnc-standalone-server tigervnc-common -y

sudo apt install xfce4 xfce4-goodies -y

sudo apt install autocutsel -y

---

# 2. Konfigurasi ~/.vnc/xstartup

#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS

autocutsel -fork
autocutsel -selection PRIMARY -fork

exec dbus-launch --exit-with-session startxfce4

chmod +x ~/.vnc/xstartup

---

# 3. Menjalankan VNC

vncserver :1

Port VNC
5901

Cek session

vncserver -list

Stop session

vncserver -kill :1

---

# 4. Install noVNC

git clone https://github.com/novnc/noVNC.git

cd noVNC/utils

./novnc_proxy --listen 6080 --vnc localhost:5901

Akses

http://SERVER_IP:6080/vnc.html

---

# 5. Clipboard Browser

Harus menjalankan autocutsel

autocutsel -fork
autocutsel -selection PRIMARY -fork

---

# 6. Service VNC

/etc/systemd/system/vncserver@.service

[Unit]
Description=TigerVNC Server
After=network.target

[Service]
Type=forking
User=user
Group=user
WorkingDirectory=/home/user

PIDFile=/home/user/.vnc/%H:%i.pid

ExecStart=/usr/bin/vncserver :%i
ExecStop=/usr/bin/vncserver -kill :%i

Restart=on-failure

[Install]
WantedBy=multi-user.target

Reload

sudo systemctl daemon-reload

Enable

sudo systemctl enable vncserver@1

---

# 7. Service noVNC

/etc/systemd/system/novnc.service

[Unit]
Description=noVNC Web Client
After=network.target vncserver@1.service
Requires=vncserver@1.service

[Service]
Type=simple
User=user
Group=user
WorkingDirectory=/home/user/noVNC

ExecStart=/home/user/noVNC/utils/novnc_proxy --listen 6080 --vnc localhost:5901

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

Reload

sudo systemctl daemon-reload

Enable

sudo systemctl enable novnc

---

# 8. Port

VNC
5901

noVNC
6080

---

# 9. Akses Browser

http://SERVER_IP:6080/vnc.html
