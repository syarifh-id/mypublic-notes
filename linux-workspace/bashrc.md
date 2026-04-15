```bash
# 1. PROTEKSI NON-INTERAKTIF
[[ $- != *i* ]] && return

# 2. SETTING HISTORY (Optimasi & Gabungan)
HISTCONTROL=ignoreboth
HISTSIZE=1000
HISTFILESIZE=2000
shopt -s histappend
# Sinkronisasi history antar terminal secara real-time
export PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"

# 3. SETTING SHELL & TERMINAL
shopt -s checkwinsize
export TERM=xterm-256color
export COLORTERM=truecolor
[ -x /usr/bin/lesspipe ] && eval "$(SHELL=/bin/sh lesspipe)"

# 4. ALIASES
alias ls='ls --color=auto'
alias grep='grep --color=auto'
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
alias ip='ip -c a'
alias alert='notify-send --urgency=low -i "$([ $? = 0 ] && echo terminal || echo error)" "$(history|tail -n1|sed -e '\''s/^\s*[0-9]\+\s*//;s/[;&|]\s*alert$//'\'')"'

# Muat file alias eksternal jika ada
[[ -f ~/.bash_aliases ]] && . ~/.bash_aliases

# 5. PATH & ENVIRONMENT VARIABLES
# Gabungkan PATH agar tidak berulang-ulang memanggil export
export ANDROID_HOME="$HOME/Android/Sdk"
export ANDROID_SDK_ROOT="$ANDROID_HOME"
export LLAMA="$HOME/Tools/llama.cpp/build/bin"

export PATH="$PATH:$HOME/go/bin:$ANDROID_HOME/emulator:$ANDROID_HOME/platform-tools:$HOME/.local/kitty.app/bin"
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$LLAMA"

# Load Cargo/Rust
[[ -f "$HOME/.cargo/env" ]] && . "$HOME/.cargo/env"

# 6. PENAMPILAN (PROMPT)
# Karena kamu pakai oh-my-posh, tidak perlu setting PS1 manual atau parse_git_branch lagi
if [ -f "/home/altair/.local/bin/oh-my-posh" ]; then
    eval "$(/home/altair/.local/bin/oh-my-posh init bash --config ~/.poshthemes/kushal.omp.json)"
fi

# Warna tambahan untuk folder yang 'writable' oleh orang lain
export LS_COLORS="$LS_COLORS:ow=37;41:"
export GCC_COLORS='error=01;31:warning=01;35:note=01;36:caret=01;32:locus=01:quote=01'

# 7. COMPLETION
if ! shopt -oq posix; then
    [[ -f /usr/share/bash-completion/bash_completion ]] && . /usr/share/bash-completion/bash_completion
fi

```
