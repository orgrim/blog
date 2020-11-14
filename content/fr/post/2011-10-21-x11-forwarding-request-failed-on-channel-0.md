---
title: X11 forwarding request failed on channel 0
date: "2011-10-21"
categories:
- NetBSD
tags:
- netbsd
- X11
- ssh
---

Quand j'essaye de me logguer sur ma box NetBSD fraichement passé en
X.org modular, j'ai ça :

    
    orgrim@serfouette ~ $ ssh rateau 
    X11 forwarding request failed on channel 0
    Last login: Fri Oct 21 12:20:24 2011 from serfouette.home.orgrim.net
    

WTF? ça tombe en marche SSH normalement.

Et bien le souci vient de l'échange des magic cookies pour
l'authentification entre serveurs X à travers SSH, c'est utilisé par le
X11 forwarding et on a besoin de plusieurs choses pour ça :

-   le chemin complet vers `xauth` sur le client
-   le chemin complet vers `xauth` sur le serveur
-   Avoir `X11Forward yes` sur le serveur quand on le demande sur le
    client (il vaut `yes` dans le `/etc/ssh/ssh_config` du client par
    flemme de taper `-X` sur la ligne de commande)

Le client est sous Linux, donc pas de soucis le chemin en dur
`/usr/bin/xauth` dans le binaire `sshd` marche. Par contre pour NetBSD
avec du X.org modular, il faut décommenter l'option `XAuthLocation` dans
`/etc/ssh/sshd_config` et `/etc/ssh/ssh_config` pour donner le bon
chemin vers `xauth`, comme indiqué en commentaire dans ces deux
fichiers :

    
    # If you use xorg from pkgsrc then uncomment the following line.
    #  XAuthLocation /usr/pkg/bin/xauth
    

Enfin, comme le `X11Forward` est à `no` par défaut sur NetBSD, je l'ai
activé. La **vrai solution** est de faire ça côté client en utilisant
explicitement l'option `-X` quand on veut ouvrir des fenêtres sur le
serveur.

Le pire c'est que Google ne sort rien sur le message "X11 forwarding
request failed on channel 0", à part une question sans réponse sur un
stackoverflow-like en Russe !
