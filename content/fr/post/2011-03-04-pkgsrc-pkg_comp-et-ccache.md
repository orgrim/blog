---
title: pkgsrc, pkg_comp et ccache
date: "2011-03-04"
categories:
- NetBSD
tags:
- netbsd
- pkgsrc
- pkg_comp
- ccache
---

Pour utiliser ccache dans un chroot `pkg_comp`, on commence par installer
ccache :

    
    # pkg_comp build devel/ccache
    # pkg_add /usr/pkgsrc/packages/All/ccache-3.1.4.tgz
    

En utilisant la cible `package-install` dans le chroot, ccache s'y
trouve installé. On l'installe aussi sur le système pour surveiller les
statistiques plus tard.

Ensuite, on édite le `etc/mk.conf` du chroot, par exemple
`/local/pkg_comp/default/etc/mk.conf`, pour y définir les variables
suivantes :

    
    # ...
    # fin de la conf speciale pkg_comp
    
    CCACHE_DIR=${WRKOBJDIR}/.ccache
    PKGSRC_COMPILER = ccache gcc
    

On créé ensuite le répertoire du cache, si le chemin du chroot est
`/local/pkg_comp/default`, avec la variable `WRKOBJDIR` laissée par
défaut :

    
    # mkdir /local/pkg_comp/default/pkg_comp/obj/pkgsrc/.ccache
    

Ensuite, il suffit de compiler ses packages comme d'habitude.

Enfin, on peut suivre les statistiques du cache avec la commande
suivante :

    
    # CCACHE_DIR=/local/pkg_comp/default/pkg_comp/obj/pkgsrc/.ccache ccache -s
    cache directory                     /local/pkg_comp/default/pkg_comp/obj/pkgsrc/.ccache
    cache hit                            133
    cache miss                          3053
    called for link                      383
    compile failed                        43
    preprocessor error                    34
    autoconf compile/link                388
    unsupported compiler option          216
    no input file                         55
    files in cache                      6201
    cache size                          58.7 Mbytes
    max cache size                    1024.0 Mbytes
    

P.S. : Une doc pour mettre en place un chroot `pkg_comp` est disponible
[sur le wiki](http://wiki.orgrim.net/netbsd/pkgsrc/pkg_comp)

