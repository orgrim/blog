---
title: Dual boot Debian/NetBSD
date: "2010-11-29"
categories:
- NetBSD
tags:
- netbsd
- debian
- dual-boot
---

L'installation d'un dual boot Linux/BSD est somme toute assez simple, on
coupe le disque en deux et on installe les systèmes l'un après l'autre.
La seule difficulté reste sur l'installation/configuration des
bootloaders.

<!--more-->

Voici comment j'ai installé mon laptop en dual boot Debian/NetBSD.
D'abord, quelques principes/astuces:

-   Linux s'installe dans différentes partitions DOS
-   NetBSD installe ses partitions à l'intérieur d'une partition DOS qui
    lui est dédiée
-   On peut partager la swap entre les deux systèmes
-   On installe d'abord Linux et on utilise du LVM pour éviter de créer
    des partitions étendues
-   GRUB sera le boot loader principal et on chainload'era le boot
    loader de NetBSD

Table des partitions DOS:

-   Part 1: /boot (100Mb) - type Linux
-   Part 2: swap (taille de la RAM) - type Swap
-   Part 3: LVM (moitié du reste) - type Linux LVM
-   Part 4: NetBSD (le reste) - type NetBSD

Etapes:

1.  Installer Debian (on prend une squeeze avec grub2)
2.  Installer NetBSD
3.  Booter NetBSD et lancer:
    `installboot -v /dev/rwd0a /usr/mdec/bootxx_ffsv1` (wd0a est la
    partition / et commence au début de la partition DOS allouée à
    NetBSD)
4.  Rebooter sur le mode Rescue de Debian:
    1.  Choisir son root fs et executer un shell dedans
    2.  Monter tous les FS: `mount -t ext3 -a`
    3.  Editer `/etc/grub.d/40_custom` (voir plus bas) :
    4.  Lancer: `grub-install`

5.  Rebooter et vérifier que tout est bien accessible

`/etc/grub.d/40_custom` :

{% highlight bash %}
#!/bin/sh
exec tail -n +3 $0
menuentry "NetBSD" {
      set root=(hd0,4)
      chainloader +1
}
```
