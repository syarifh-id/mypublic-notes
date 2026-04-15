```bash
# ======================
# WINDOW & LAYOUT
# ======================

background_opacity 0.75
background_blur 8
#dynamic_background_opacity 1

window_padding_width 4 12
hide_window_decorations no

# Ukuran (opsional, bisa dihapus kalau pakai remember)

remember_window_size yes


# ======================
# FONT
# ======================

font_family JetBrains Mono, CaskaydiaCove Nerd Font Mono, Cascadia Mono, Consolas, Courier New, monospace
bold_font JetBrains Mono Bold
italic_font JetBrains Mono Italic
font_size 10.0
letter_spacing 0
line_height 0.9
# ======================
# ENVIRONMENT
# ======================

env TERM=xterm-256color
env COLORTERM=truecolor


# ======================
# CURSOR & BEHAVIOR
# ======================

cursor_shape block
cursor_blink_interval 0
cursor_trail 1
scrollback_lines 2000
enable_audio_bell no
resize_debounce_time 0


# keymap

enabled_layouts tall,splits,grid
initial_window_layout tall


map ctrl+shift+h launch --location=vsplit


map ctrl+shift+left neighboring_window left
map ctrl+shift+right neighboring_window right
map ctrl+shift+up neighboring_window up
map ctrl+shift+down neighboring_window down

# BEGIN_KITTY_THEME
# Dainty Dark
include current-theme.conf
# END_KITTY_THEME

```
