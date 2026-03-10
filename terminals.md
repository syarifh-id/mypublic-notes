### Custom terminals

22:45 altair: ~ ➤ 
```
PS1='\[\e[36m\]\A\[\e[0m\] \[\e[1;32m\]\u\[\e[0m\]: \[\e[0;37m\]\W\[\e[0m\] \[\e[0m\]➤ '
```

[22:45] altair: ~ ➤ 
```
PS1='[\[\e[36m\]\A\[\e[0m\]] \[\e[1;32m\]\u\[\e[0m\]: \[\e[0;37m\]\W\[\e[0m\] ➤ '
```

### Activate contom bash Only on tmux 

```
if [ -n "$TMUX" ]; then
    PS1=...
fi
```




### Enable true color for helix theme
```
export TERM=xterm-256color
export COLORTERM=truecolor
```
### bash custom color 

```
#Permissions Color
export LS_COLORS="$LS_COLORS:ow=37;41:"

```
git branch

```
# Git branch
parse_git_branch() {
    git rev-parse --abbrev-ref HEAD 2>/dev/null
}

PS1='\[\e[36m\]\A\[\e[0m\] \[\e[1;32m\]\u\[\e[0m\]: \[\e[0;37m\]\W\[\e[0m\] \e[1;34m\]$(parse_git_branch)\[\e[0m\]➤ '


```

### Bash history Suggestion


```
touch ~/.inputrc
```

```
nano ~/.inputrc

"\e[A": history-search-backward
"\e[B": history-search-forward

```
reload
```
bind -f ~/.inputrc
```
to use, type command and press CTRL+r
