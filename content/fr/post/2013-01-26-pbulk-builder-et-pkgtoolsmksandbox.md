---
title: pbulk-builder et pkgtools/mksandbox
date: "2013-01-26"
categories:
- NetBSD
tags:
- netbsd
- pkgsrc
---

Après la réinstall d'une de mes machines de build en 6.0.1, j'ai eu la
bonne surprise de voir que le script de build `pbulk-builder`
([https://github.com/orgrim/nb-utils]) ne trouvait plus `mksandbox`
dans l'arbre de pkgsrc. Il est désormais dans son paquet :
`pkgtools/mksandbox`

<!--more-->

Le script est à jour. le script de montage de la sandbox se prend un
`sed` dans la foulée pour éviter qu'il force le montage de l'arbre
pkgsrc dans `/tree/pkgsrc`...

[https://github.com/orgrim/nb-utils]: https://github.com/orgrim/nb-utils
