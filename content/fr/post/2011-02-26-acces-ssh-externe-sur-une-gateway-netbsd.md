---
title: Accès SSH externe sur une gateway NetBSD
date: "2011-02-26"
categories:
- NetBSD
tags:
- netbsd
- ssh
- pf
---

Pour permettre l'accès à la gateway depuis Internet avec un maximum de
sécurité, j'ai configuré le serveur OpenSSH (fournit dans basesys) et PF
pour :

-   N'autoriser l'accès que par clé publique
-   Empêcher les attaques par force brute sur le serveur

La configuration d'OpenSSH peut se faire de deux façon :

1.  On désactive l'authentification par mot de passe et l'utilisation de
    PAM pour que la méthode keyboard-ineractive ne laisse rien passer
2.  On désactive l'authentification par mot de passe et le
    challenge/respone pour garder l'utilisation de PAM

J'ai choisi la 2ème méthode qui permet de garder la fonctionnalité de
pam\_nologin (seul root peut se logguer si `/etc/nologin` existe, en
modifiant `/etc/ssh/sshd_config` :

    
    PermitRootLogin no
    PasswordAuthentication no
    ChallengeResponseAuthentication no
    UsePam yes
    

Ainsi, l'authentification par mot de passe est totalement désactivée.

Pour empêcher les attaques par force brute sur le serveur, on configure
PF de cette façon :

-   Il faut une table pour enregistrer les IP à bannir
-   On utilise l'option `max-src-conn-rate` sur la règle qui ouvre le
    port de SSH : pas plus de deux connexions dans un intervalle de dix
    secondes par IP.
-   On ajoute une règle pour bloquer les IP présentes de la table

Les règles PF, à placer dans `/etc/pf.conf` :

    
    table <ssh_bans> persist
    
    # allow ssh on the external interface, limit to connections in ten seconds
    pass in on $ext_if inet proto tcp to ($ext_if) port ssh \\
        keep state (max-src-conn-rate 2/10, overload <ssh_bans> flush)
    
    # reject banned ip from connectiong to ssh port
    block return in on $ext_if inet proto tcp from <ssh_bans> to ($ext_if) port ssh
    

Pour surveiller, on utilise les options de manipulation des tables de
`pfctl` :

    
    # pfctl -t ssh_bans -T show
    # pfctl -t ssh_bans -T add X.Y.Z.T
    # pfctl -t ssh_bans -T delete X.Y.Z.T
    

Enfin, un petit thread qui explique à quoi sert le challenge/response et la méthode keyboard-interactive : [http://groups.google.com/group/comp.security.ssh/browse_thread/thread/c483316ac6abcb74](http://groups.google.com/group/comp.security.ssh/browse_thread/thread/c483316ac6abcb74).
