---
title: pkgin adventures - utiliser pkgin avec pkgsrc-current
date: "2011-03-12"
categories:
- NetBSD
tags:
- netbsd
- pkgin
---

**pkgin** est un outil de gestion de paquets binaires pour pkgsrc, le
système de paquets de NetBSD. Pour pouvoir l'utiliser il faut donc des
paquets binaires, sauf que les binaires ne sont officiellement
disponibles que pour les releases trimestrielles de pkgsrc. Quand on
suit pkgsrc-current, il faut donc compiler les paquets et fabriquer un
dépôt.

<!--more-->

La solution consiste donc à utiliser l'équipe habituelle pour compiler
les paquets sans gêner le système : `pkg_comp` et `pkg_chk`. Pour le
dépôt on a simplement besoin d'un serveur web pour les mettre à
disposition.

Voici un petit résumé de la procédure :

​1. A partir d'une machine ayant l'ensemble de ses paquets déjà
installés, on met en place un chroot `pkg_comp` comme indiqué [sur le
wiki][].

​2. On génère la liste des paquets à construire à partir des paquets
installé avec `pkg_chk` :

    
    # pkg_chk -g
    

​3. On les compile dans le chroot :

    
    # pkg_comp build pkgtools/pkg_chk
    # pkg_comp chroot pkg_chk -ua
    

​4. On génère le fichier `pkg_summary` qui va bien :

    
    # cd /usr/pkgsrc/packages/All
    # pkg_info -X *.tgz | bzip2 > pkg_summary.bz2
    

​5. On ajoute le définition du dépôt local dans la configuration de
`pkgin`, en éditant `/usr/pkg/etc/pkgin/repositories.conf` :

    
    file:///usr/pkgsrc/packages/All
    

On peut enfin utiliser `pkgin` pour ajouter et supprimer des paquets. Il
suffit de regénérer le `pkg_summary` à chaque nouvelle compilation de
paquet dans le chroot.

Pour l'upgrade, il faut pouvoir ne garder que les paquets les plus à
jour dans le dépôt, pour cela l'outil `pkg_tarup` entre en jeu, il
permet de générer les paquets binaires à partir de l'installation
courante.

Après une upgrade avec `pkg_chk` dans le chroot, on peut mettre à jour
le dépôt :

    
    # pkg_comp build pkgtools/pkg_tarup
    # rm /usr/pkgsrc/packages/All/*
    # pkg_comp chroot pkg_tarup -a -d /usr/pkgsrc/packages/All \\'*\\'
    # cd /usr/pkgsrc/packages/All
    # pkg_info -X *.tgz | bzip2 > pkg_summary.bz2
    

Enfin, pour mettre à jour avec `pkgin` :

    
    # pkgin up
    # pkgin fug
    

[sur le wiki]: http://wiki.orgrim.net/netbsd/pkgsrc/pkg_comp
