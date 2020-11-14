---
title: pkgin adventures - conflit résolu
date: "2011-03-12"
categories:
- NetBSD
tags:
- netbsd
- pkgin
---

Il y a quelques jours, il a été décidé de remplacer `libungif` par
`giflib` dans pkgsrc. Pour éviter de mixer les deux et donc avoir des
problèmes, les deux paquets se déclarent mutuellement en conflit. A
partir de maintenant la dépendance par défaut est sur `giflib`, ce qui a
donc fait que ma mise à jour (`pkg_chk` dans un `pkg_comp`) a tellement
buté dessus que j'ai décidé qu'il serait plus simple de repartir d'un
chroot `pkg_comp` tout neuf...

<!--more-->

Même si j'avais oublié de retirer `libungif` de mon
`/usr/pkgsrc/pkgchk.conf` au début, j'ai bien obtenu un dépôt tout neuf
pour `pkgin`.

Et là, la magie de `pkgin` a opéré :

    
    # pkgin up
    processing local summary...
    updating database: 100%
    downloading pkg_summary.bz2:    0Bbps 100%
    processing remote summary (file:///usr/pkgsrc/packages/amd64/All)...
    updating database: 100%
    
    # pkgin fug
    calculating dependencies... done.
    giflib-4.1.6 (to be installed) conflicts with installed package libungif-4.1.4nb1.
    proceed ? [y/N] n
    
    # pkgin srd libungif
    local reverse dependency tree for libungif
            feh-1.3.4nb8
            giblib-1.2.4nb9
            python-mode-1.0nb1
            php-mode-1.4.0nb1
            mplayer-1.0rc20100913nb5
            emacs-23.2nb4
            imlib2-1.4.2nb6
    
    # pkgin rm libungif
    8 packages to delete: mplayer-1.0rc20100913nb5 php-mode-1.4.0nb1 python-mode-1.0nb1
     feh-1.3.4nb8 emacs-23.2nb4 giblib-1.2.4nb9 imlib2-1.4.2nb6 libungif-4.1.4nb1
    proceed ? [y/N] y
    ....
    
    # pkgin in feh emacs php-mode python-mode mplayer
    ...
    
    # pkgin fug
    calculating dependencies... done.
    
    21 packages to be upgraded: epdfview-0.1.7nb10 mercurial-1.8 scmgit-base-1.7.3.5
    scmgit-docs-1.7.3.5 poppler-glib-0.16.2 libgnome-2.32.0nb2 libgnomeui-2.24.4nb2
    poppler-glib-0.16.2 poppler-utils-0.16.2 t1lib-5.1.2nb1 gtk2+-2.22.1nb1
    tex-dvipdfm-0.13.2dnb3 curl-7.21.3 glib2-2.26.1nb2 gnutls-2.10.4 libksba-1.1.0
    dialog-1.1.20110118 libidn-1.19 luatex-0.65.0nb1 web2c-2010nb6 poppler-0.16.2
    
    21 packages to be installed: poppler-0.16.3 dialog-1.1.20110302 libidn-1.20
    luatex-0.65.0nb2 web2c-2010nb7 curl-7.21.4 glib2-2.28.2 gnutls-2.10.5nb1 libksba-1.2.0
    gtk2+-2.24.1 tex-dvipdfm-0.13.2dnb4 poppler-glib-0.16.3 libgnome-2.32.1
    libgnomeui-2.24.5 poppler-glib-0.16.3 poppler-utils-0.16.3 t1lib-5.1.2nb2
    epdfview-0.1.7nb11 mercurial-1.8.1 scmgit-base-1.7.4.1 scmgit-docs-1.7.4.1
    (40M to download, 292M to install)
    
    proceed ? [y/N] y
    ...
    

Et hop, résolution du conflit à la main, certes, mais <ins>très
facilement</ins> et avec <ins>un seul outil</ins>. Sans `pkgin`,
j'aurais du itérer à coup de `pkg_info -R`, `pkg_delete` et
`pkg_chk -ub`... Parce `pkgin` ressort tout l'arbre des dépendances :

    
     # pkg_info -R giflib
    Information for giflib-4.1.6:
    
    Required by:
    imlib2-1.4.2nb7
    emacs-23.2nb5
    mplayer-1.0rc20100913nb6
    
    
    # pkgin srd giflib
    local reverse dependency tree for giflib
            feh-1.3.4nb8
            giblib-1.2.4nb9
            python-mode-1.0nb1
            php-mode-1.4.0nb1
            mplayer-1.0rc20100913nb6
            emacs-23.2nb5
            imlib2-1.4.2nb7
    

