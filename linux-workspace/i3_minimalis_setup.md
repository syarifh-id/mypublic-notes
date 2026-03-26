# Setup i3 Minimalis Gelap + Transparansi (Kali Linux)

Dokumentasi langkah demi langkah konfigurasi i3 dan Alacritty agar tampil minimalis, gelap, dan modern dengan transparansi ringan.

---

# 1. Install Paket yang Dibutuhkan

```bash
sudo apt update
sudo apt install fonts-jetbrains-mono feh picom rofi alacritty
```

---

# 2. Backup Config Sebelum Mengubah

```bash
cp ~/.config/i3/config ~/.config/i3/config.backup
```

Restore jika terjadi error:

```bash
cp ~/.config/i3/config.backup ~/.config/i3/config
```

---

# 3. Config i3 Minimalis Modern

Edit file:

```bash
nano ~/.config/i3/config
```

Contoh konfigurasi sederhana dan clean:

```bash
set $mod Mod4

font pango:JetBrains Mono 10

# Terminal & Launcher
set $term alacritty
bindsym $mod+Return exec $term
bindsym $mod+d exec rofi -show drun

# Kill window
bindsym $mod+Shift+q kill

# Focus (Vim-style)
bindsym $mod+h focus left
bindsym $mod+j focus down
bindsym $mod+k focus up
bindsym $mod+l focus right

# Move window
bindsym $mod+Shift+h move left
bindsym $mod+Shift+j move down
bindsym $mod+Shift+k move up
bindsym $mod+Shift+l move right

# Layout
bindsym $mod+f fullscreen toggle
bindsym $mod+w layout tabbed
bindsym $mod+s layout stacking
bindsym $mod+e layout toggle split

# Workspace
set $ws1 "1"
set $ws2 "2"
set $ws3 "3"
set $ws4 "4"

bindsym $mod+1 workspace $ws1
bindsym $mod+2 workspace $ws2
bindsym $mod+3 workspace $ws3
bindsym $mod+4 workspace $ws4

bindsym $mod+Shift+1 move container to workspace $ws1
bindsym $mod+Shift+2 move container to workspace $ws2
bindsym $mod+Shift+3 move container to workspace $ws3
bindsym $mod+Shift+4 move container to workspace $ws4

# Border & Gaps
default_border pixel 2
gaps inner 8
gaps outer 5
hide_edge_borders smart

# Warna Dark
client.focused          #89b4fa #89b4fa #1e1e2e #89b4fa
client.unfocused        #313244 #313244 #cdd6f4 #313244
client.focused_inactive #45475a #45475a #cdd6f4 #45475a

# Autostart
exec --no-startup-id feh --bg-scale /usr/share/backgrounds/kali/kali-dark.jpg
exec --no-startup-id picom
```

Reload:

```bash
Mod + Shift + c
```

Restart jika perlu:

```bash
Mod + Shift + r
```

---

# 4. Config Picom (Transparansi)

Buat file:

```bash
mkdir -p ~/.config/picom
nano ~/.config/picom/picom.conf
```

Isi stabil (VM-safe):

```ini
backend = "xrender";
vsync = false;

shadow = true;
shadow-radius = 12;
shadow-opacity = 0.3;


active-opacity = 1.0;
```

Tes manual jika perlu:

```bash
picom --backend xrender --vsync false &
```

---

# 5. Config Alacritty Modern (TOML)

Lokasi:

```bash
~/.config/alacritty/alacritty.toml
```

Contoh config modern dark:

```toml
[window]
opacity = 0.85
padding = { x = 12, y = 10 }
decorations = "none"

[font]
normal = { family = "JetBrains Mono", style = "Regular" }
bold = { family = "JetBrains Mono", style = "Bold" }
italic = { family = "JetBrains Mono", style = "Italic" }
size = 11.0

[env]
TERM = "alacritty"
COLORTERM = "truecolor"

[colors.primary]
background = "#0f111a"
foreground = "#cdd6f4"

[colors.normal]
black   = "#1b1e28"
red     = "#f38ba8"
green   = "#a6e3a1"
yellow  = "#f9e2af"
blue    = "#89b4fa"
magenta = "#cba6f7"
cyan    = "#94e2d5"
white   = "#cdd6f4"

[colors.bright]
black   = "#585b70"
red     = "#f38ba8"
green   = "#a6e3a1"
yellow  = "#f9e2af"
blue    = "#89b4fa"
magenta = "#cba6f7"
cyan    = "#94e2d5"
white   = "#ffffff"
```

Tutup semua Alacritty lalu buka ulang.

---

# 6. Perbaikan Bug Input (Jika btop Tidak Bisa Mengetik)

Tambahkan di ~/.bashrc:

```bash
stty -ixon
```

Logout lalu login ulang.

---

# 7. Troubleshooting

## Cek syntax i3

```bash
i3 -C
```

## Cek picom berjalan

```bash
pgrep picom
```

## Cek versi Alacritty

```bash
alacritty --version
```

---

# Hasil Akhir

- Dark minimal theme
- Border tipis + gaps modern
- Transparansi ringan
- Font coding-friendly
- Cocok untuk workflow terminal-heavy dan security testing

