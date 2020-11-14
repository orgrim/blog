---
title: pkgin adventures - bien générer son pkg_summary
date: "2011-03-12"
categories:
- NetBSD
tags:
- netbsd
- pkgin
---

Comme indiqué dans d'autres posts, j'abuse des chroots `pkg_comp` pour
tenir mes paquets à jour. Je suis récemment passé à l'utilisation de
`pkgin` pour la gestion de mes paquets une fois préparés dans le chroot.

<!--more-->

`pkgin` se base sur `pkg_summary` pour connaître toutes les informations
des paquets, nécessaires à sa popotte. Il y a plusieurs façons de créer
un fichier `pkg_summary` à donner à `pkgin`, mais seule une façon
fonctionne correctement :

​1. On génère le fichier à partir des paquets déjà installés :

    
    # pkg_info -a -X | bzip2 > pkg_summary.bz2
    

​2. On génère le fichier à partir des tarballs présentes dans
`/usr/pkgsrc/packages/All` :

    
    # cd /usr/pkgsrc/packages/All
    # pkg_info -X *.tgz | bzip2 > pkg_summary.bz2
    

La méthode 1 n'est pas valable car l'information sur les tarballs
manque. Ainsi, `pkgin` considère les tailles de tarball à 0 comme
valables, ce qui arrive lorsqu'un dépôt est injoignable : le fetch
laisse un fichier vide dans le cache que `pkgin` considère comme
correct.

Il faut donc utiliser la méthode 2 pour fournir l'information correcte à
`pkgin`.

Pour conclure, l'investigation autour de ce souci, a permis aux
développeurs du projet d'ajouter :

-   Un mode verbose pour avoir plein d'informations utiles
-   Un message d'avertissement lorsque `pkgin` rencontre un paquet à
    avec une taille à 0
