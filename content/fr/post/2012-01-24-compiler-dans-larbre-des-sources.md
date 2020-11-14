---
title: Compiler dans l'arbre des sources
date: "2012-01-24"
categories:
- NetBSD
tags:
- netbsd
- build
---

Je m'occupe actuellement de préparer ma box pour le FOSDEM, et il
s'avère qu'il manque le support du DRM (Direct Rendering Manager, le
truc pour avoir de l'accélération graphique dans le kernel pour X.org)
pour ma carte vidéo. Il s'agit de NetBSD 5.1.1, la version 6 n'aura pas
ce manque.

<!--more-->

Il faut donc recompiler un noyau pour ajouter cette fonctionnalité, pour
faire vite on ne passe pas par `build.sh`, les tools etc, on compile
directement dans l'arbre des sources, montrer comment faire ça est le
but de post.

On commence par récupérer les sources sur un serveur CVS près de chez
soi :

    
    # export CVS_RSH=ssh
    # export CVSROOT="anoncvs@anoncvs.NetBSD.org:/cvsroot"
    # cd /usr
    # cvs -q -z3 co -P -rnetbsd-5-1-1-RELEASE src
    

Ensuite, on n'a pas besoin d'avoir une conf particulière dans son
`/etc/mk.conf`, on n'a juste à ajouter les options dans le fichier de
conf du kernel et compiler directement par l'intermédiaire de `make`.

On édite le fichier `/usr/src/sys/arch/i386/conf/GENERIC.local`, pour y
ajouter les lignes suivantes, il est inclus par le fichier `GENERIC` :

    
    # DRI driver    
    i915drm*        at vga?         # Intel i915, i945 DRM driver
    mach64drm*      at vga?         # mach64 (3D Rage Pro, Rage) DRM driver
    mgadrm*         at vga?         # Matrox G[24]00, G[45]50 DRM driver
    r128drm*        at vga?         # ATI Rage 128 DRM driver
    radeondrm*      at vga?         # ATI Radeon DRM driver
    savagedrm*      at vga?         # S3 Savage DRM driver
    sisdrm*         at vga?         # SiS DRM driver
    tdfxdrm*        at vga?         # 3dfx (voodoo) DRM driver
    

On compile le kernel à la main :

    
    # config GENERIC
    # cd ../compile/GENERIC
    # make depend
    # make
    

On installe le kernel à la main :

    
    # mv /netbsd /netbsd.old
    # cp netbsd /
    # chmod 444 /netbsd
    

Pour revenir facilement en arrière en cas de souci, on peut ajouter la
ligne suivante dans `/boot.cfg`, en deuxième position :

    
    menu=Boot old kernel:boot netbsd.old
    

Il ne reste plus qu'à rebooter sur le nouveau kernel.

Référence : [Le guide]

On peut faire la même manip pour mettre à jour une partie du système
seulement, par exemple lorsqu'une faille de sécurité doit être corrigée.
La méthode de compilation dans l'arbre des sources est indiquée dans
l'avis.

Plus généralement, la méthode est la suivante, avec l'exemple de `ls` :

    
    # cd /usr/src/bin/ls
    # make USETOOLS=no cleandir
    # make USETOOLS=no dependall
    

Le binaire résultant et sa doc sont prêts dans le répertoire courant, il
ne reste qu'à installer :

    
    # make USETOOLS=no install
    
[Le guide]: http://www.netbsd.org/docs/guide/en/chap-kernel.html#chap-kernel-building-manually
