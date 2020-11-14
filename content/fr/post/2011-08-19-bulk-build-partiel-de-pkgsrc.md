---
title: Bulk build partiel de pkgsrc
date: "2011-08-19"
categories:
- NetBSD
tags:
- netbsd
- pkgsrc
---

En suivant l'[excellent tip de Mr GuiGui2][], j'ai pu monter ma petite
archi de bulk build personnelle pour fournir du package tout frais à
[pkgin][].

J'ai donc ajouté le bloc magique suivant à mon `/etc/mk.conf`, qui
permet de gérer la présence de commentaires dans `pkgchk.conf` :

    
    # bulk build config
    DEPENDS_TARGET= bulk-install
    BATCH=          yes
    
    BULK_PREREQ+=   pkgtools/lintpkgsrc
    .if defined(SPECIFIC_PKGS)
    PKGLIST!=               awk '$$1 !~ /^\\#/ {print $$1}' ${PKGCHK_CONF}
    .       for _pkg_ in ${PKGLIST}
    HOST_SPECIFIC_PKGS+=    ${_pkg_}
    .       endfor
    .endif
    

Pour aller plus loin, j'ai automatisé le process au maximum pour lancer
des bulk build réguliers par cron, grâce au script [bulk-builder][]. Ce
script remplace `do-sandbox-build` et `do-sandbox-upload`, il est
également capable de gérer des chemins alternatifs, mettre à jour
`pkgsrc` avant de lancer le bulk.

La procédure pour mettre ça en place est donc :

​1. Ajouter le bloc montré plus haut à `/etc/mk.conf` et y définir
`PKGCHK_CONF`, il s'agit du chemin vers une liste de packages au format
\\"catetgorie/package\\", un par ligne, qu'on peut automatiquement créer
avec `pkg_chk -g`.

​2. Créer la sandbox :

    
    # cd /usr/pkgsrc/mk/bulk
    # sh mksandbox --without-x /usr/sandbox
    

​3. Créer et configurer `/usr/pkgsrc/mk/bulk/build.conf` :

    
    # cd /usr/pkgsrc/mk/bulk
    # cp build.conf-example build.conf
    # vi build.conf
    

​4. Lancer le bulk build :

    
    # sh bulk-builder -u -R
    

-   `-u` demande de `cvs up` le répertoire `/usr/pkgsrc` avant de
    commencer
-   `-R` demande de ne pas uploader le résultat (les packages)

Enfin, il suffit d'utiliser la ligne suivante pour utiliser les packages
avec `pkgin`, dans `/usr/pkg/etc/pkgin/repositories.conf` :

    
    file:///usr/pkgsrc/packages/All
    

</p>

[excellent tip de Mr GuiGui2]: http://www.guigui2.net/index.php?post/2011/07/30/sudo-sh-/usr/pkgsrc/mk/bulk/do-sandbox-build-s
[pkgin]: http://pkgin.net
