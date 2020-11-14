---
title: Un bulk build par jour dans un screen
date: "2013-05-10"
categories:
- NetBSD
tags:
- netbsd
- pkgsrc
---

Mes packages NetBSD sont préparés par pbulk, qui tourne en continu grâce
au script [pbulk-builder][].

<!--more-->

J'avais prévu avec l'option `-1` de lui éviter d'entrer dans une boucle
infinie, et j'ai pas eu tort. En effet, la majorité des tours ne fait
que mettre à jour l'arbre [pkgsrc][], résoudre les dépendances, sans
rien compiler de nouveau. La première solution que j'ai trouvé a été de
créer une règle sieve pour ne pas recevoir des dizaines de mails de
rapport de bulk inutiles, en les plaçant dans un répertoire séparé...

N'ayant jamais pris le temps d'utiliser cette fameuse option one-shot,
j'ai décidé de mettre le lancement du bulk dans la crontab, sauf que
c'est long et qu'il vaut mieux suivre ça avec screen. C'est possible
grâce aux options `-d` et `-m` (avec `-S` pour mettre un titre) :

    
    0 23 * * * /usr/pkg/bin/screen -dmS bulk -c /root/.screenrc /usr/pkg/bin/bash ~orgrim/nb-utils/pkgsrc/pbulk-builder -c -1 /data/pbulk
    

On peut alors attacher le screen quand le bulk tourne.

PS: merci au fans de tmux de passer sur le chan \#netbsdfr sur Freenode
pour me dire comment faire pareil avec tmux :)


[pbulk-builder]: https://github.com/orgrim/nb-utils/blob/master/pkgsrc/pbulk-builder
[pkgsrc]: http://pkgsrc.org
