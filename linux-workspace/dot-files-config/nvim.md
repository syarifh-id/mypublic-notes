```md
# Catatan Setup & Make Over Neovim

## 1. Tujuan
Melakukan “make over” Neovim agar:
- Tampilan modern
- Ada plugin (UI, explorer, search)
- Autocomplete & LSP aktif
- Lebih produktif seperti VSCode

---

## 2. Struktur Dasar

File utama:
```

~/.config/nvim/init.lua

````

---

## 3. Install Plugin Manager

Menggunakan `lazy.nvim`:

```bash
git clone https://github.com/folke/lazy.nvim ~/.local/share/nvim/lazy/lazy.nvim
````

Tambahkan ke `init.lua`:

```lua
vim.opt.rtp:prepend("~/.local/share/nvim/lazy/lazy.nvim")
```

---

## 4. Konfigurasi Lengkap `init.lua`

```lua
-- =====================
-- Basic settings
-- =====================
vim.g.mapleader = " "
vim.opt.number = true
vim.opt.relativenumber = true
vim.opt.termguicolors = true

-- =====================
-- Load lazy.nvim
-- =====================
vim.opt.rtp:prepend("~/.local/share/nvim/lazy/lazy.nvim")

-- =====================
-- Plugins
-- =====================
require("lazy").setup({

  -- Theme
  {
    "folke/tokyonight.nvim",
    config = function()
      vim.cmd("colorscheme tokyonight")
    end
  },

  -- Statusline
  {
    "nvim-lualine/lualine.nvim",
    config = function()
      require("lualine").setup()
    end
  },

  -- File explorer
  {
    "nvim-tree/nvim-tree.lua",
    config = function()
      require("nvim-tree").setup()
      vim.keymap.set("n", "<leader>e", ":NvimTreeToggle<CR>")
    end
  },

  -- Telescope
  {
    "nvim-telescope/telescope.nvim",
    dependencies = { "nvim-lua/plenary.nvim" },
    config = function()
      local telescope = require("telescope.builtin")
      vim.keymap.set("n", "<leader>ff", telescope.find_files)
      vim.keymap.set("n", "<leader>fg", telescope.live_grep)
    end
  },

  -- Treesitter
  {
    "nvim-treesitter/nvim-treesitter",
    build = ":TSUpdate",
    config = function()
      require("nvim-treesitter.configs").setup({
        highlight = { enable = true }
      })
    end
  },

  -- LSP (API baru)
  {
    "neovim/nvim-lspconfig",
    config = function()
      vim.lsp.config("lua_ls", {})
      vim.lsp.enable("lua_ls")

      -- keybinding LSP
      vim.keymap.set("n", "gd", vim.lsp.buf.definition)
      vim.keymap.set("n", "K", vim.lsp.buf.hover)
      vim.keymap.set("n", "<leader>rn", vim.lsp.buf.rename)
      vim.keymap.set("n", "<leader>ca", vim.lsp.buf.code_action)
    end
  },

  -- Autocomplete
  {
    "hrsh7th/nvim-cmp",
    dependencies = {
      "hrsh7th/cmp-nvim-lsp",
      "hrsh7th/cmp-buffer",
      "hrsh7th/cmp-path"
    },
    config = function()
      local cmp = require("cmp")

      cmp.setup({
        mapping = cmp.mapping.preset.insert({
          ["<C-Space>"] = cmp.mapping.complete(),
          ["<CR>"] = cmp.mapping.confirm({ select = true }),
          ["<Tab>"] = cmp.mapping.select_next_item(),
          ["<S-Tab>"] = cmp.mapping.select_prev_item(),
        }),

        sources = {
          { name = "nvim_lsp" },
          { name = "buffer" },
          { name = "path" },
        }
      })
    end
  }

})
```

---

## 5. Install LSP (lua-language-server)

### Linux (contoh)

```bash
sudo apt install lua-language-server
```

Atau manual:

```bash
wget https://github.com/LuaLS/lua-language-server/releases/latest/download/lua-language-server-*.tar.gz
tar -xzf lua-language-server-*.tar.gz
cd lua-language-server-*
chmod +x bin/lua-language-server
sudo ln -s $(pwd)/bin/lua-language-server /usr/local/bin/lua-language-server
```

Cek:

```bash
lua-language-server --version
```

---

## 6. Menjalankan Neovim

```bash
nvim
```

Plugin akan otomatis terinstall.

---

## 7. Keybinding

### Leader Key

```
Space
```

---

### Dasar Neovim

| Key | Fungsi        |
| --- | ------------- |
| i   | insert mode   |
| Esc | normal mode   |
| :w  | save          |
| :q  | keluar        |
| :wq | save & keluar |

---

### Navigasi

| Key     | Fungsi   |
| ------- | -------- |
| h j k l | gerak    |
| gg      | ke atas  |
| G       | ke bawah |

---

### File Explorer

| Key       | Fungsi         |
| --------- | -------------- |
| Space + e | toggle sidebar |
| Enter     | buka file      |
| a         | tambah file    |
| d         | delete         |
| r         | rename         |

---

### Telescope

| Key        | Fungsi    |
| ---------- | --------- |
| Space + ff | cari file |
| Space + fg | cari teks |

---

### Autocomplete

| Key          | Fungsi  |
| ------------ | ------- |
| Ctrl + Space | trigger |
| Tab          | next    |
| Shift + Tab  | prev    |
| Enter        | confirm |

---

### LSP

| Key        | Fungsi           |
| ---------- | ---------------- |
| gd         | go to definition |
| K          | hover doc        |
| Space + rn | rename           |
| Space + ca | code action      |

---

## 8. Fitur yang Didapat

* Theme modern
* Sidebar file explorer
* Fuzzy search
* Syntax highlight canggih
* Autocomplete
* LSP (intellisense)
* Keybinding efisien

---

## 9. Masalah yang Ditemui

### Error:

```
require('lspconfig') is deprecated
```

### Solusi:

Gunakan API baru:

```lua
vim.lsp.config("lua_ls", {})
vim.lsp.enable("lua_ls")
```

---

## 10. Catatan Penting

* Semua plugin harus di dalam:

```lua
require("lazy").setup({...})
```

* Jangan buat table `{}` di luar setup
* Pastikan koma `,` tidak kurang
* Pastikan LSP terinstall di sistem

---

```
```

