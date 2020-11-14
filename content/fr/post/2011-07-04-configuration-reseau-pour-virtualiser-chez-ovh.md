---
title: Configuration réseau pour virtualiser chez OVH
date: "2011-07-04"
categories:
- Unix
tags:
- unix
- debian
- lxc
- ipv6
---

Sur mon serveur chez OVH, j'ai un ensemble de machines virtuelles KVM et
(bientôt) de conteneurs LXC. Pour fournir du réseau à tout ce petit
monde, j'utilise de l'IPv4 et de l'IPv6, voici comment c'est configuré.

Pour l'IPv4, on a un nombre limité d'IP publiques parce que ça vaut de
la thune et que ça va être de plus en tendu de multiplier les adresses,
il nous faut un réseau privé (beurk), du NAT (rebeurk) et des
redirections à base d'iptables (re-rebeurk). Il nous faut surtout un
bridge, c'est une Debian donc ça se passe dans
`/etc/network/interfaces` :

    
    auto lo
    iface lo inet loopback
    
    auto eth0
    iface eth0 inet manual
    
    auto br0
    iface br0 inet static
            bridge_ports eth0
            bridge_fd 0
            bridge_maxwait 0
            address 188.165.205.xxx
            netmask 255.255.255.0
            network 188.165.205.0
            broadcast 188.165.205.255
            gateway 188.165.205.254
            post-up iptables -t nat -A POSTROUTING -s 10.42.0.0/24 -o br0 -j SNAT --to 188.165.205.xxx
            pre-down iptables -t nat -D POSTROUTING -s 10.42.0.0/24 -o br0 -j SNAT --to 188.165.205.xxx
    
    iface br0 inet6 static
            address 2001:41D0:xxxx:96ce::1
            netmask 64
    
    auto br0:0
    iface br0:0 inet static
            address 10.42.0.1
            netmask 255.255.255.0
    

Pour le bridge, on met l'IPv4 publique et une règle iptable pour NAT-er
ce qui sort. Le réseau privé est 10.42.0.0/24, on peut bien sûr choisir
ce qu'on veut dans ce que fourni la RFC 1918.

Pour l'IPv6, chez OVH c'est bizarre : il fournissent un prefix 64 bit et
avec une route accessible sur le /56 incluant, mais pas à partir du /64.
Il faut donc ruser de la façon suivante niveau routage.

On assigne l'IPv6 avec le prefix /64, c'est plus propre, sinon le kernel se plaint avec /56 :

    
    # ip -6 addr add 2001:41D0:xxxx:96ce::1/64 dev br0
    
    
On met une route pour atteindre la gateway dans le /56 :

    
    # ip -6 route add 2001:41d0:xxxx:96ce::/56 dev br0
    

On met la route par défaut :

    
    # ip -6 route add default via 2001:41d0:xxxx:96ff:ff:ff:ff:ff
    

Pour avoir cette configuration appliquée automatiquement, il faut créer
`/etc/network/if-up.d/ovh-ipv6-route` :

    
    #!/bin/sh
    
    if [ "$IFACE" = "br0" ]; then
        ip -6 route add 2001:41d0:2:96ce::/56 dev br0
        ip -6 route add default via 2001:41d0:2:96ff:ff:ff:ff:ff
    fi
    

Et pour l'arrêt, `/etc/network/if-down.d/ovh-ipv6-route` :

    
    #!/bin/sh
    
    if [ "$IFACE" = "br0" ]; then
        ip -6 route del default via 2001:41d0:2:96ff:ff:ff:ff:ff
        ip -6 route del 2001:41d0:2:96ce::/56 dev br0
    fi
    

Il faut ensuite activer le forward entre interfaces, sinon pas de
routage vers les VM, dans `/etc/sysctl.conf` :

    
    net.ipv6.conf.all.forwarding=1
    

Enfin, le routage chez OVH ne permet pas d'intercaler un routeur pour
découper le /64, il faut donc ruser avec du proxy NDP (Neighbor
Discovery Protocol).

On active le proxy NDP, dans `/etc/sysctl.conf` :

    
    net.ipv6.conf.all.proxy_ndp=1
    

On ajoute les IP à proxiser, celle de la gateway et chacune des IP
    fournies aux VM :

    
    # ip neigh add proxy 2001:41d0:xxxx:96ff:ff:ff:ff:ff dev br0
    # ip neigh add proxy 2001:41d0:xxxx:96ce::2 dev br0
    

En cas de reboot d'une VM, il faut d'abord virer ses IP du proxy, puis
les remettre, sinon le kernel de la VM se plein d'une duplication
d'adresse :

    
    # ip neigh del proxy 2001:41d0:xxxx:96ce::2 dev br0
    -> reboot de la VM
    # ip neigh add proxy 2001:41d0:xxxx:96ce::2 dev br0
    

