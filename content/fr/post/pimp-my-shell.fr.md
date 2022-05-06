---
title: "Pimp My Shell"
date: 2022-05-06T16:26:01+02:00
categories:
- Unix
- Debian
tags: 
- shell
- tuning
- fonts
---

Voici quelques notes pour plus tard sur la configuration et tuning de mon shell
/ desktop sur Debian Bullseye (stable au moment de l'écriture de ce post).

### fonts

Installables par les paquets Debian :

* Font basique : `fonts-dejavu`
* Fonts Arial, Bitstream Vera etc : `ttf-mscorefonts-installer`
* Emoji : `fonts-noto-color-emoji`, `fonts-symbola`

On peut aussi ajouter les fonts pour avoir les caractères des alphabets
différents du latin, pour le spam c'est sympa, mais facultatif.

Icônes pour `lsd`, via les Nerd Fonts
(<https://www.nerdfonts.com/font-downloads>) :

```
mkdir -p ~/.local/share/fonts/nerd-fonts
cd ~/.local/share/fonts/nerd-fonts
curl -LO https://github.com/ryanoasis/nerd-fonts/releases/download/v2.1.0/DejaVuSansMono.zip
unzip DejaVuSansMono.zip
rm DejaVuSansMono.zip
fc-cache -v
```

### rxvt-unicode

Configuration de `~/.Xdefaults` pour choisir les bonnes fonts et couleurs :

```
URxvt*background: #000000
URxvt*foreground: #ffffff
URxvt*tintColor: #000000
URxvt*shading: 17
URxvt*font: xft:DejaVuSansMono Nerd Font Mono:style=Regular:size=9,xft:Symbola,xft:Noto Color Emoji:style=Regular:size=9
URxvt*letterSpace: -1
```

### prompt et bash-git-prompt

On garde le prompt du shell light, sur une seule ligne, vert en user simple :

``` shell
PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\W\[\033[00m\] \$ '
```

et on ajoute `bash-git-prompt`, pour avoir des informations utiles lorsque
qu'on est dans un dépôt git :

```
git clone https://github.com/magicmonty/bash-git-prompt.git ~/.bash-git-prompt
```

et dans `~/.bashrc` :

``` shell
case "$TERM" in
xterm*|rxvt*)
    GIT_PROMPT_ONLY_IN_REPO=1
    GIT_PROMPT_START='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\W\[\033[00m\]'
    GIT_PROMPT_END=' \$ '
    source ~/.bash-git-prompt/gitprompt.sh
    ;;
*)
    ;;
esac
```

### bat

un clone de `cat(1)` avec de la pagination, de la coloration syntaxique, des
numéros de lignes, etc.

```
sudo apt-get install bat
alias cat='bat'
```

### lsd

un clone de `ls(1)` avec plus de couleurs, des icônes, etc. Pour l'installer,
on utilise le paquet Debian fournit, vu qu'il n'est pas dans les dépôts de
Debian :

```
curl -LO https://github.com/Peltoche/lsd/releases/download/0.21.0/lsd_0.21.0_amd64.deb
sudo dpkg -i lsd_0.21.0_amd64.deb
sudo apt-get -f install
```

```
alias ls='lsd'
```

### splatmoji

Menu de selection d'Emoji :

```
curl -LO https://github.com/cspeterson/splatmoji/releases/download/v1.2.0/splatmoji_1.2.0_all.deb
sudo dpkg -i splatmoji_1.2.0_all.deb
sudo apt-get -f install
```

Ajouter le wrapper proposé pour gérer le copier-coller selon le terminal ou le
navigateur, dans `~/bin/splatmoji-wrap` :

``` shell
#!/bin/bash

# You can figure out which window properties you want to look for using `xprop`under X 
WINDOWNAME="$(xdotool getwindowfocus getwindowname)"
case "${WINDOWNAME}" in
    *Firefox*|*Brave*)
        exec splatmoji copypaste ;;
    *)
        exec splatmoji type ;;
esac
```

Ne pas oublier : `chmod 755 ~/bin/splatmoji-wrap`

Ajouter le raccourci à i3 :

```
bindsym $mod+slash exec "splatmoji-wrap"
```

### meteo

Pour avoir la météo locale :

```
alias meteo='curl wttr.in/${ville}'
```

### cal

Pour avoir des semaines qui commencent le lundi avec `cal` :

```
alias cal='ncal -Mb'
```

### du et numfmt

Pour trier et afficher les tailles de répertoire dans l'ordre human-readable,
une fonction :

``` shell
function sdu() { du -csB1 "$@" | sort -n | numfmt --to=si; }
```
