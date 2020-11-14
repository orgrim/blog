---
title: Montrer les dépendances avec make dans pkgsrc
date: "2011-08-18"
categories:
- NetBSD
tags:
- netbsd
- pkgsrc
---

Généralement, on peut savoir quelles sont les dépendances d'un package
en utilisant `make show-depends`, mais cela ne montre que les
dépendances pour l'installation, les dépendances pour la compilation ne
sont pas montrées.

<--more-->
    
    $ cd /usr/pkgsrc/databases/postgresql90-server/
    $ make show-depends
    postgresql90-client>=9.0.4:../../databases/postgresql90-client
    

Pour connaître les dépendances selon leur type (installation ou
compilation), on peut utiliser la cible `show-depends-pkgpaths` alliée à
la variable `DEPENDS_TYPE`.

Pour avoir seulement les dépendances de compilation :

    
    $ make DEPENDS_TYPE=build show-depends-pkgpaths
    devel/bison
    devel/gmake
    pkgtools/digest
    

Pour avoir seulement celles d'installation :

    
    $ make DEPENDS_TYPE=install show-depends-pkgpaths
    databases/postgresql90-client
    

Et enfin pour montrer les deux types, qui est aussi le comportement par
défaut :

    
    $ make DEPENDS_TYPE=all show-depends-pkgpaths
    databases/postgresql90-client
    devel/bison
    devel/gmake
    pkgtools/digest
    
    $ make show-depends-pkgpaths
    databases/postgresql90-client
    devel/bison
    devel/gmake
    pkgtools/digest
    

Pour plus d'information, le Makefile qui contrôle cette cible est
`mk/bsd.utils.mk`.

</p>

