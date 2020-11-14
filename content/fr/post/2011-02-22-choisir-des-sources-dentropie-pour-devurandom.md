---
title: Choisir des sources d'entropie pour /dev/[u]random
date: "2010-12-30"
categories:
- NetBSD
tags:
- netbsd
- random
- rndctl
---

Pour monter un service DNS on a besoin de `/dev/random` pour générer la
clé rndc. Sans collecte d'entropie, `/dev/random` ne crache rien.

<!--more-->

La preuve sur un domU Xen en NetBSD 5.1 (le dom0 suit 5-stable), donnée par
`ktruss` :

    
    fourche ~ # ktruss rndc-confgen -a -t /var/chroot/named
    [...]
       519      1 rndc-confgen open("/dev/random", 0x4, 0) = 3
    [...]
       519      1 rndc-confgen read(0x3, 0xbf7fe4a8, 0x10) Err#35 EAGAIN
    

Le `read()` attend gentiment que `/dev/random` lui fournisse du nombre
aléatoire pendant que celui-ci dit de revenir plus tard. La raison est
simple, aucune source d'entropie n'est configurée pour la collecte sur
le domU, comme l'indique `rndctl` :

    
    fourche ~ # rndctl -l
    Source                 Bits Type      Flags
    xennet1                   0 net  
    xennet0                   0 net  
    xbd0                      0 disk
    

Et les stats ne sont pas glorieuses :

    
    fourche ~ # rndctl -s    
                   51 bits mixed into pool
                    0 bits currently stored in pool (max 4096)
                    0 bits of entropy discarded due to full pool
                   51 hard-random bits generated
                16749 pseudo-random bits generated
    

Après une lecture du man de rndctl, on active tout :

    
    fourche ~ # rndctl -ce -d xennet1
    fourche ~ # rndctl -ce -d xennet0
    fourche ~ # rndctl -ce -d xbd0
    

Ça va tout de suite mieux, `rndc-confgen` termine, et on a des « bits »
qui commencent à remplir le « pool » :

    
    fourche ~ # rndctl -ls
    Source                 Bits Type      Flags
    xennet1                   0 net  estimate, collect
    xennet0                  80 net  estimate, collect
    xbd0                      0 disk estimate, collect
                  133 bits mixed into pool
                   80 bits currently stored in pool (max 4096)
                    0 bits of entropy discarded due to full pool
                   53 hard-random bits generated
                17003 pseudo-random bits generated
    

Pour avoir ça activé au boot, dans `/etc/rc.conf` :

    
    rndctl=YES
    rndctl_flags="xbd0; -c -t net" # Voir les commentaires dans /etc/rc.d/rndctl
    

Reste à découvrir pourquoi ce n'est pas activé par défaut en domU...

</p>

