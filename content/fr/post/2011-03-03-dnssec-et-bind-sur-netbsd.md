---
title: DNSSEC et Bind sur NetBSD
date: "2011-03-03"
categories:
- NetBSD
tags:
- netbsd
- bind
- dnssec
---

En reconfigurant ma gateway sur NetBSD, le serveur Bind fournit dans
« basesys » refusait de fonctionner à cause de l'activation de DNSSEC
par défaut.

<!--more-->

Pour le désactiver, il suffit de modifier les paramètres
suivants dans `/etc/named.conf` :

    
    options {
            # ...
    
            dnssec-enable no;
            dnssec-validation no;
    #        dnssec-lookaside auto;
    
            # ...
    };
    

Mais ce n'est pas la solution idéale, sachant que DNSSEC est activé sur
la zone root depuis juin 2010, autant l'utiliser. Pour avoir DNSSEC dans
le Bind du basesys, il manque juste la configuration des clés. On
récupère la DNSKEY de la zone root avec la commande qui va bien :

    
    # dig +dnssec +multiline . dnskey > /chemin/vers/root_dnskey
    

On obtient la clé qu'il faut vérifier, on génère donc l'enregistrement
DS correspondant :

    
    # dnssec-dsfromkey -2 -f /chemin/vers/root_dnskey .
    . IN DS 19036 8 2 49AAC11D7B6F6446702E54A1607371607A1A41855200FD2CE1CDDE32 F24E8FB5
    

**Attention :** le dernier espace doit être ignoré.

On compare donc à ce que fournit l'IANA
([http://data.iana.org/root-anchors/root-anchors.xml](http://data.iana.org/root-anchors/root-anchors.xml)) et d'autres
comme Kirei ([http://www.kirei.se/en/2010/06/20/root-ksk/](http://www.kirei.se/en/2010/06/20/root-ksk/)) qui
fournissent également les signatures PGP à vérifier.

Quand c'est bon (GPG dit OK), on ajoute un bloc `managed-keys` avec le
contenu de l'enregistrement DNSKEY de la zone root dans
`/etc/named.conf` :

    
    options {
            # ...
    
            dnssec-enable yes;
            dnssec-validation yes;
            dnssec-lookaside auto;
            managed-keys-directory "keys";
    
            # ...
    };
    
    managed-keys {
            "." initial-key 257 3 8 "AwEAAagAIKlVZrp...";
    };
    

Pour bénéficier du « look aside », c'est à dire une autre source de
vérification des données du DNS, il faut ajouter la clé sur service DLV
de ISC (l'organisation qui code Bind), en récupérant la clé là :
[https://www.isc.org/solutions/dlv](https://www.isc.org/solutions/dlv) et après vérification des
signatures PGP, on peut ajouter un bloc `trusted-keys` à
`/etc/named.conf` :

    
    trusted-keys {
            dlv.isc.org. 257 3 5 "BEAAAAP...";
    };
    

Avant de relancer le service, on peut configurer des logs (voir le
[wiki](http://wiki.orgrim.net/netbsd/named#configurer-les-logs) pour les logs dans le chroot) pour mettre du debug sur la
partie DNSSEC :

    
    logging {
            # ...
            channel dnssec_log_file {
                    file "/var/log/dnssec.log" versions 3 size 20m;
                    severity debug 3;
                    print-category yes;
                    print-severity yes;
                    print-time yes;
            };
    
            category dnssec { dnssec_log_file; };
            # ...
    };
    

Et corriger quelques permissions :

    
    # mkdir /etc/namedb/keys
    # chown named:named /etc/namedb/keys
    

Enfin, on peut redémarrer le service et surveiller les logs :

    
    # /etc/rc.d/named restart
    

Pour tester, on utilise `dig` sur zone signée :

    
    # dig +dnssec +multiline net
    
    ; <<>> DiG 9.7.2-P3 <<>> +dnssec +multiline net
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5595
    ;; flags: qr rd ra ad; QUERY: 1, ANSWER: 0, AUTHORITY: 4, ADDITIONAL: 1
    
    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags: do; udp: 4096
    ;; QUESTION SECTION:
    ;net.                   IN A
    
    ;; AUTHORITY SECTION:
    net.                    518 IN SOA a.gtld-servers.net. nstld.verisign-grs.com. (
                                    1299183189 ; serial
                                    1800       ; refresh (30 minutes)
                                    900        ; retry (15 minutes)
                                    604800     ; expire (1 week)
                                    86400      ; minimum (1 day)
                                    )
    net.                    518 IN RRSIG SOA 8 1 900 20110310201309 (
                                    20110303200309 3980 net.
                                    TUwUOfzcgGN/lXERU2Y+l3xMr8h6cg/t7ODOiGh8wSNq
                                    8zOEvaTYeR+aKY76mY9X1d+odTcHv52ewLs0nQLlvzFb
                                    iz48fxJrWoNKz5D1HxYDJGNqAsxh3usX0xxnNQoM0cIm
                                    vvw5uFWxFK8cJ0Xha1s4Rpd2z2gVse3yZB3kU78= )
    A1RT98BS5QGC9NFI51S9HCI47ULJG6JH.net. 518 IN RRSIG NSEC3 8 2 86400 20110310193846 (
                                    20110303192846 3980 net.
                                    IhkKHupik3uQMq94xJQqaLmTeXPCeROIdcicFRh5ocsP
                                    Mzfm/sO+9MpPdPpffCeCg3TtPxHhln6N0ffUa5jNoNnK
                                    kmxZTqF6OUPnW7+pUm2kIysZxjOR5wK4n40IyTj8QNWZ
                                    DspTJTVV7v/4RHgqnoHo2vHcZvLR744Y8PDEBH4= )
    A1RT98BS5QGC9NFI51S9HCI47ULJG6JH.net. 518 IN NSEC3 1 1 0 - A25R64HGRKT76GSK0JS1PNJ44MEELOJ6 NS SOA RRSIG DNSKEY NSEC3PARAM
    
    ;; Query time: 12 msec
    ;; SERVER: 127.0.0.1#53(127.0.0.1)
    ;; WHEN: Thu Mar  3 21:20:00 2011
    ;; MSG SIZE  rcvd: 511
    
    

Le flag **ad** indique que la réponse est valide selon DNSSEC.

On voit des trucs du genre dans les logs :

    
    03-Mar-2011 18:22:12.734 dnssec: debug 3: validating @0xba5c7000: net DS: starting
    03-Mar-2011 18:22:12.735 dnssec: debug 3: validating @0xba5c7000: net DS: attempting positive response validation
    03-Mar-2011 18:22:12.738 dnssec: debug 3: validating @0xba5ca000: . NS: starting
    03-Mar-2011 18:22:12.739 dnssec: debug 3: validating @0xba5ca000: . NS: attempting insecurity proof
    03-Mar-2011 18:22:12.739 dnssec: debug 3: validating @0xba5ca000: . NS: insecurity proof failed
    03-Mar-2011 18:22:12.740 dnssec: info: validating @0xba5ca000: . NS: got insecure response; parent indicates it should be secure
    03-Mar-2011 18:22:12.740 dnssec: debug 3: validator @0xba5ca000: dns_validator_destroy
    03-Mar-2011 18:22:12.778 dnssec: debug 3: validating @0xba5ca000: . DNSKEY: starting
    03-Mar-2011 18:22:12.779 dnssec: debug 3: validating @0xba5ca000: . DNSKEY: attempting insecurity proof
    03-Mar-2011 18:22:12.779 dnssec: debug 3: validating @0xba5ca000: . DNSKEY: insecurity proof failed
    03-Mar-2011 18:22:12.779 dnssec: info: validating @0xba5ca000: . DNSKEY: got insecure response; parent indicates it should be secure
    03-Mar-2011 18:22:12.779 dnssec: debug 3: validator @0xba5ca000: dns_validator_destroy
    03-Mar-2011 18:22:12.791 dnssec: debug 3: validating @0xba5ca000: . NS: starting
    03-Mar-2011 18:22:12.792 dnssec: debug 3: validating @0xba5ca000: . NS: attempting positive response validation
    03-Mar-2011 18:22:12.988 dnssec: debug 3: validating @0xba5c4000: . DNSKEY: starting
    03-Mar-2011 18:22:12.988 dnssec: debug 3: validating @0xba5c4000: . DNSKEY: attempting positive response validation
    03-Mar-2011 18:22:13.002 dnssec: debug 3: validating @0xba5c4000: . DNSKEY: verify rdataset (keyid=19036): success
    03-Mar-2011 18:22:13.002 dnssec: debug 3: validating @0xba5c4000: . DNSKEY: signed by trusted key; marking as secure
    03-Mar-2011 18:22:13.002 dnssec: debug 3: validator @0xba5c4000: dns_validator_destroy
    03-Mar-2011 18:22:13.003 dnssec: debug 3: validating @0xba5ca000: . NS: in fetch_callback_validator
    03-Mar-2011 18:22:13.003 dnssec: debug 3: validating @0xba5ca000: . NS: keyset with trust 8
    03-Mar-2011 18:22:13.003 dnssec: debug 3: validating @0xba5ca000: . NS: resuming validate
    03-Mar-2011 18:22:13.006 dnssec: debug 3: validating @0xba5ca000: . NS: verify rdataset (keyid=21639): success
    03-Mar-2011 18:22:13.006 dnssec: debug 3: validating @0xba5ca000: . NS: marking as secure, noqname proof not needed
    03-Mar-2011 18:22:13.006 dnssec: debug 3: validator @0xba5ca000: dns_validator_destroy
    03-Mar-2011 18:22:13.008 dnssec: debug 3: validating @0xba5c7000: net DS: in fetch_callback_validator
    03-Mar-2011 18:22:13.008 dnssec: debug 3: validating @0xba5c7000: net DS: keyset with trust 8
    03-Mar-2011 18:22:13.008 dnssec: debug 3: validating @0xba5c7000: net DS: resuming validate
    03-Mar-2011 18:22:13.011 dnssec: debug 3: validating @0xba5c7000: net DS: verify rdataset (keyid=21639): success
    03-Mar-2011 18:22:13.011 dnssec: debug 3: validating @0xba5c7000: net DS: marking as secure, noqname proof not needed
    03-Mar-2011 18:22:13.011 dnssec: debug 3: validator @0xba5c7000: dns_validator_destroy
    

On voit bien, que lors de la vérification de la signature de *net*, Bind
commence par vérifier *.* (la zone root) et que ça marche.

PS : ne pas oublier de (re)mettre la verbosité des logs à *info* après
coup.

[http://data.iana.org/root-anchors/root-anchors.xml]: http://data.iana.org/root-anchors/root-anchors.xml
[http://www.kirei.se/en/2010/06/20/root-ksk/]: http://www.kirei.se/en/2010/06/20/root-ksk/
[https://www.isc.org/solutions/dlv]: https://www.isc.org/solutions/dlv
[wiki]: http://wiki.orgrim.net/netbsd/named#configurer-les-logs
