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
    PS1='\[\e[36m\]\A\[\e[0m\] \[\e[1;32m\]\u\[\e[0m\]: \[\e[0;37m\]\W\[\e[0m\] \[\e[0m\]➤ '
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
