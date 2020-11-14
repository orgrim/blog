---
title: NetBSD en KVM
date: "2011-07-04"
categories:
- NetBSD
tags:
- netbsd
- kvm
---

Comme je viens d'investir dans un [serveur kimsufi] (le 16G), je me suis dis qu'avoir
quelques machines NetBSD pour servir la bonne cause ça serait bien cool.

<!--more-->

En fait, j'ai deux besoins :

-   Avoir un dépôt de paquet et de quoi fait des bulk build réduits de
    pkgsrc pour me permettre de n'utiliser uniquement l'excellent pkgin
-   Fournir des machines à la Build Farm de PostgreSQL, parce qu'il n'y
    a même pas de machine NetBSD en i386 et en amd64 (seulement powerpc
    et mips)

J'ai donc décidé de monter 2 machines NetBSD en KVM sur ma grosse box
Debian, qui fait tourner ça grâce à KVM.

Pour l'installation, il y a 2 possibilités, soit on fait du VNC, soit on
redirige la sortie VGA dans du curses. Pour le disque j'ai choisi de
poser directement les données sur des volumes logiques, dans ce cas, il
faut désactiver le cache ce qui permet une infime perte de puissance en
I/O.

Il faut commencer par récupérer l'ISO d'installation :

    
    wget ftp://ftp.fr.netbsd.org/pub/NetBSD/iso/5.1/amd64cd-5.1.iso
    

Ensuite, créer le volume logique (on a choisi de donner 50 Go) et lancer
l'installation, en curses ça passe nickel, il faut choisir d'utiliser la
console serie :

    
    kvm -drive file=/dev/system/kvm-nb64-d1,cache=none \\
        -m 1024 \\
        -net nic,model=e1000 -net tap \\
        -name nb64 \\
        -curses \\
        -cdrom /home/orgrim/netbsd/amd64cd-5.1.iso \\
        -boot d \\
        -k fr
    

Une fois installé, on lance la commande suivante dans un screen, on
demande à KVM de fournir l'accès console en série dans un fichier, ce
qui permet d'avoir la console QEMU disponible directement dans le
screen :

    
    kvm -nographic \\
        -drive file=/dev/system/kvm-nb64-d1,cache=none \\
        -m 1024 \\
        -net nic,model=e1000,macaddr=DE:AD:BE:EF:37:D1 -net tap \\
        -name nb64 \\
        -boot c \\
        -serial unix:/tmp/nb64.sock,server,nowait
    

Note: merci de changer la MAC de l'interface réseau, c'est utilisé en
prod chez moi :)

Enfin, il est important de démarrer le noyau NetBSD sans ACPI ni SMP (en
mettant le defaut à 4 dans `/boot.cfg`)

Pour accéder à la machine en console série :

    
    minicom -D unix#/tmp/nd64.sock
    

Si on a oublié de choisir la console série, on peut l'activer de cette
façon :

1. Booter avec `-curses -k fr` à la place de `-nographic`
2. Lancer la commande suivante pour activer la console série : 
    
        # installboot -v -o console=com0,speed=19200 /dev/rwd0a /usr/mdec/bootxx_ffsv1
        File system:         /dev/rwd0a
        Primary bootstrap:   /usr/mdec/bootxx_ffsv1
        Boot options:        timeout 5, flags 0, speed 19200, ioaddr 0, console com0
        
    

[serveur kimsufi]: http://www.kimsufi.com/fr/
