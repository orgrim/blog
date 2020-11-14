---
title: Le client de la BuildFarm de PostgreSQL dans pkgsrc-wip
date: "2011-08-06"
categories:
- NetBSD
tags:
- netbsd
- pkgsrc
- postgresql
---

Comme j'annonçais précédemment, je contribue deux machines NetBSD à la
BuildFarm de PostgreSQL. La compilation ne se fait automagiquement
qu'après la configuration du client (écrit en Perl). Il n'est d'ailleurs
pas forcément très convi à installer, c'est pourquoi je l'ai packagé
pour pkgsrc : [http://pkgsrc.se/wip/pgbuildfarm][].

<!--more-->

En espérant qu'il soit ajouté à l'arbre officiel...

Voici la configuration pour lancer des builds sur NetBSD, dans
`/usr/pkg/etc/pgbuildfarm/build-farm.conf`, en commençant par le chemin
du miroir du dépôt Git :

    
    # Modifier dans %conf
    scmrepo => '/usr/pgbuildfarm/pgsql-base.git',
    

Le client est destiné à être lancé par cron, connu pour son
environnement light, c'est pourquoi les paramètres d'environnement
doivent être adaptés :

-   Le make GNU s'appelle gmake chez nous
-   Pas mal de programmes proviennent de pkgsrc, il faut donc que le
    client ait `/usr/pkg/bin` dans son PATH, et puisse trouver les
    bibliothèques issues des packages.

    
    make => 'gmake',
    aux_path => "/usr/pkg/bin",
    
    build_env =>
    {
        PATH => "/usr/pkg/bin:$ENV{PATH}",
        LD_LIBRARY_PATH => "/usr/pkg/lib",
    },
    
    config_env =>
    {
        CC => 'gcc',
        PATH => "/usr/pkg/bin:$ENV{PATH}",
        LD_LIBRARY_PATH => "/usr/pkg/lib",
    },
    

Enfin le plus important, les options du `configure`, la plupart
nécessitent des packages supplémentaires comme `python` ou la `libxml`.
Ce qui est primordial ici est d'utiliser le « template » NetBSD :

    
    config_opts =>
    [qw(
        --enable-cassert
        --enable-debug
        --enable-nls
        --enable-integer-datetimes
        --with-perl
        --with-python
        --with-tcl
        --with-krb5
        --with-includes=/usr/include/krb5:/usr/pkg/include
        --with-libraries=/usr/pkg/lib
        --with-openssl
        --with-template=netbsd
        --enable-thread-safety
    )],
    

Pour toutes ces options, les packages suivants ont été installés :

-   `devel/bison`
-   `devel/flex`
-   `lang/python26` (et `pkgtools/pkg_alternatives` pour avoir le lien
    python)
-   `lang/perl5`
-   `lang/tcl`
-   `textproc/libxml2`
-   `textproc/libxslt`
-   `devel/readline`

P.S. : Il n'y a que les particularités de NetBSD décrites ici, en
complément du [wiki de PostgreSQL][].


[http://pkgsrc.se/wip/pgbuildfarm]: http://pkgsrc.se/wip/pgbuildfarm
[wiki de PostgreSQL]: http://wiki.postgresql.org/wiki/PostgreSQL_Buildfarm_Howto
