---
title: modular-xorg, radeon et pas de KMS
date: "2013-08-23"
categories:
- NetBSD
tags:
- netbsd
- X11
- pkgsrc
---

Il y avait un moment que je n'avais pas touché à NetBSD et donc mis à
jour mon lappy avec du pkg frais. Entre temps, la version de X.org
modular, donc issue de pkgsrc, est revenue en 2012, avec son lot de
drivers mis à jour. Le drivers xf86-video-ati, est passé en version
7.1.0, sauf qu'il fonctionne uniquement en KMS (Kernel Mode Setting),
chose qu'on n'a pas encore dans notre kernel.

<!--more-->

Il faut donc la dernière version de la branche 6 du driver, qui contient
encore le support UMS, disponible dans le paquet `x11/xf86-video-ati6`,
qui porte le même nom de package, bizarrement. Tout ça est suivi dans le
[PR 47935][].

Pour que le serveur X trouve le device avec ce driver, j'ai du virer la
ligne `BusID` dans la section Device du `xorg.conf`.


[PR 47935]: http://gnats.netbsd.org/cgi-bin/query-pr-single.pl?number=47935
