---
title: Passer de X.org natif à modular
date: "2011-09-19"
categories:
- NetBSD
tags:
- netbsd
- pkgsrc
---

X.org est fourni dans le basesys et dans pkgsrc, on appelle le premier
« native » et le second « modular » selon la valeur de la variable
`X11_TYPE` que l'on positionne dans son `/etc/mk.conf` pour signifier à
pkgsrc sur lequel linker.

<!--more-->

Il s'agit des mêmes versions à peu de choses prêt, et X.org native n'est
pas vieux ou non maintenu comme la rumeur voudrait le faire croire. Il
est juste en retard parce qu'il suit le cycle de release du basesys
alors que modular suit celui de pkgsrc et est tiré vers l'avant par les
packages qui en dépendent. Cela peut poser problème lorsqu'on suit la
cible mouvante qu'est pkgsrc-current.

La première chose à faire pour passer de native à modular est d'éditer
`/etc/mk.conf` pour changer `X11_TYPE`, on en profite pour ne plus
compiler le native :

    
    MKX11=no 
    X11_TYPE=modular
    

Ensuite, il faut modifier la liste de packages à compiler pour y ajouter
soit tout modular en installant `meta-pkgs/modular-xorg`, soit en
n'installant que le nécessaire, ça fait plus cool, dans
`/usr/pkgsrc/pkgchk.conf` (si vous avez suivi les docs ici ou dans le
wiki) :

    
    meta-pkgs/modular-xorg-apps
    meta-pkgs/modular-xorg-libs
    meta-pkgs/modular-xorg-fonts
    x11/xf86-input-keyboard
    x11/xf86-input-mouse
    x11/xf86-input-void
    x11/xf86-video-nv
    x11/xf86-video-vesa
    x11/xf86-video-vga
    x11/xf86-video-wsfb
    x11/modular-xorg-server
    

Ensuite on donne ça à manger à `pkg_comp` ou à son bulk. L'important ici
est de tout recompiler pour bien transférer les dépendances de native
vers modular : en gros on pète la sandbox, que ça soit de `mk/bulk` ou
`pkg_comp` et on recommence. Etant passé en mode bulk partiel comme
indiqué dans un précédent [post], voici comment faire :

​1. On vérifie avec `mount` que la standbox n'est pas montée ni qu'un
build tourne (dans ce cas faut le killer), sinon :

    
    # /usr/sandbox/sandbox umount
    

​2. On vérifie que le contenu des mk.conf du système et de la sandbox
sont en phase, c'est le seul fichier de la sandbox à sauver

​3. On recrée la sandbox :

    
    # rm -rf /usr/sandbox
    # cd /usr/pkgsrc/mk/bulk
    # sh mksandbox --without-x /usr/sandbox
    

​4. On nettoie les packages déjà compilés, pour forcer leur
recompilation, et les fichiers cachés du bulk build :

    
    # cd /usr/pkgsrc/packages/
    # mv CVS .CVS
    # rm -rf *
    # mv .CVS CVS
    # cd /usr/pkgsrc
    # rm .broken.html .bulk_build_id .bulk_db .bulklock .depends .dependstree .index  .order .start .supports
    # pkgclean
    

​5. On lance le bulk build et on attend, il faut prévoir entre 150 et
200 packages supplémentaires à compiler.

Les étapes suivantes sont simples, on sauvegarde tout ce qu'il faut dans
`/usr/pkg`, puis on supprime tous les packages installés :

    
    # cd /usr
    # mv pkg pkg.old
    # cd /var/db
    # mkdir old_pkgdb
    # mv pkg pkg.refcount old_pkgdb
    # rm -rf pkgin
    

A partir d'ici, on n'a plus de programmes issus de pkgsrc, en gros pas
de sudo, vim et autres...

Puis, il faut supprimer les fichiers des sets de X.org, on se base pour
cela sur le contenu de `/etc/mtree/set.x*`. On en arrive donc à un stade
où on est dans la même situation qu'après une installation du système
sans les sets de X.org natif.

Enfin, on réinstalle tout les packages avec [pkgin], qui dans sa
version 0.5.0 (du CVS) peut importer une liste de packages au format de
`pkg_chk`, qui se trouve être le même utilisé par le bulk-build, comme de
par hasard :

    
    # pkg_add http://pkgsrc.orgrim.net/NetBSD/5.1/amd64/All/pkg_install-20110805.tgz
    # mkdir -p /usr/pkg/etc
    # cp -r /usr/pkg.old/etc/pkgin /usr/pkg/etc
    # ./pkgin up
    # ./pkgin im pkgchk.conf
    

**Note :** si le `pkg_add` de pkg_install ne passe pas, essayer avec
`-f`.

et hop, reste plus qu'à reconfigurer les chemins dans
`/etc/X11/xorg.conf` et c'est bon.


[post]: /post/2011/08/19/Bulk-build-partiel-de-pkgsrc
[pkgin]: http://www.pkgin.net
