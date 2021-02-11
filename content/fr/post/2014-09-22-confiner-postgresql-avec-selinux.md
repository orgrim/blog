---
title: Confiner PostgreSQL avec SELinux
date: 2014-09-22
categories:
- PostgreSQL
tags:
- postgresql
- selinux
---

{{< notice note >}}Ce post est partiellement obsolète, avec le passage
à systemd. Voir la [mise à jour]({{< relref
"2021-02-11-postgresql-et-selinux" >}}).{{< /notice >}}

J'avais déjà expérimenté un peu avec SELinux il y a deux ans sans
aller trop loin, parce que j'entendais souvent la phrase "Si on veut
de la sécurité, il faut SELinux" et surtout à cause de l'arrivée de
l'extension `sepgsql` dans les modules contrib de PostgreSQL. Ça avait
donné une conf pour le Fosdem où finalement, j'ai plus parlé des
privilèges classiques que de SELinux.

<!--more-->

Suite à une demande au ${BOULOT}, je me suis replongé dedans, et les
choses ont peu évolué. Dans la majorité des recherches que j'ai pu
faire, SELinux reste tout de même un truc qui se met en travers la
route, c'est-à-dire, qu'il y a toujours plus de résultats sur comment
le désactiver plutôt que configurer les choses correctement. Ensuite,
certains proposent de faire aveuglement confiance à `setroubleshoot`
et `audit2allow` pour faire taire la bête. Enfin, j'ai dû trouver une
page ou deux après en deux semaines à temps plein sur le sujet qui
parlent de comment confiner un utilisateur dans le but d'implémenter ce
que promet SELinux et que toutes les entreprises demandent : le RBAC,
Role Based Access Control, ou comment donner le moins de droits
possibles aux sous-traitants qui hébergent les serveurs.

Cette fois ci, je suis allé plus loin. Déjà, je suis parti sur une
installation de CentOS 6, la famille de distribution
RHEL/CentOS/Fedora semble être la plus en avance par rapport à
SELinux : à peu près tous les services/daemons fournis dans l'install
sont confinés par SELinux. Les utilisateurs ne le sont pas au login,
il n'y a donc aucune configuration de RBAC. On verra peut-être ça plus
tard, j'ai pas encore obtenu de résultat satisfaisant sur ce sujet, et
il vaut mieux expliquer comment confiner correctement PostgreSQL avant
de passer aux roles. Tout simplement, parce qu'il est très simple de
casser ce confinement avec la configuration par défaut de RHEL, grâce
à `pg_ctl`. Enfin, l'installation des paquets du PGDG n'est pas
confinée par défaut, il manque les file contexts adaptés aux chemins
particuliers de ces paquets, prévu pour faire cohabiter plusieurs
versions majeures ; ce qui ne correspond pas ce qu'à prévu Red Hat
pour PostgreSQL.

Après cette longue introduction, mais avant de commencer, il faut
savoir administrer un minumum SELinux : si vous ne savez pas qu'il
existe des options `-Z`, ce que sont les contexts, les types et les
domaines, mieux vaut d'abord se documenter, par exemple [chez Red
Hat](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Security-Enhanced_Linux/index.html).

Voici donc les file contexts à ajouter dans un fichier *module.te* pour
confiner l'installation PostgreSQL du PGDG :

    /etc/rc\.d/init\.d/(se)?postgresql(-.*)?    --  gen_context(system_u:object_r:postgresql_initrc_exec_t,s0)
    /usr/pgsql-[0-9]+\.[0-9]+/bin/initdb        --  gen_context(system_u:object_r:postgresql_exec_t,s0)
    /usr/pgsql-[0-9]+\.[0-9]+/bin/pg_ctl        --  gen_context(system_u:object_r:postgresql_initrc_exec_t,s0)
    /usr/pgsql-[0-9]+\.[0-9]+/bin/postgres      --  gen_context(system_u:object_r:postgresql_exec_t,s0)
    /usr/pgsql-[0-9]+.[0-9]+/share/locale(/.*)?	    gen_context(system_u:object_r:locale_t,s0)
    /usr/pgsql-[0-9]+.[0-9]+/share/man(/.*)?        gen_context(system_u:object_r:man_t,s0)

Tout d'abord, pour l'init script et `pg_ctl`, on utilise le type
`postgresql_initrc_exec_t`, c'est ce qui permet de lancer PostgreSQL
dans le domaine confiné `postgresql_t` au boot, via l'init script, et
manuellement. La méthode la plus propre est de toujours utiliser
l'init script, idéalement par l'intermédiaire de `run_init` pour
démarrer, arrêter ou redémarrer le postmaster. On évite alors de
laisser trainer des choses dans `/var/{run,lock}`.

Les programmes `postgres` et `initdb` doivent avoir le type
`postgresql_exec_t` car ils exécutent le serveur PostgreSQL ; cela
doit se faire dans le domaine `postgresql_t`.

Enfin, on a placé les labels corrects sur les fichiers de traduction
et les pages de man, pour faire plus propre. Ce code source de module
SELinux alors doit être compilé et chargé.

Cette configuration est reprise dans le module SELinux disponible sur
[github]. On peut aussi l'ajouter manuellement avec `semanage` :

    semanage fcontext -a -t postgresql_initrc_exec_t '/etc/rc\.d/init\.d/(se)?postgresql(-.*)?'
    semanage fcontext -a -t postgresql_exec_t '/usr/pgsql-[0-9]+\.[0-9]+/bin/initdb'
    semanage fcontext -a -t postgresql_initrc_exec_t '/usr/pgsql-[0-9]+\.[0-9]+/bin/pg_ctl'
    semanage fcontext -a -t postgresql_exec_t '/usr/pgsql-[0-9]+\.[0-9]+/bin/postgres'
    semanage fcontext -a -t locale_t '/usr/pgsql-[0-9]+.[0-9]+/share/locale(/.*)?'
    semanage fcontext -a -t man_t '/usr/pgsql-[0-9]+.[0-9]+/share/man(/.*)?'

Il existe un certain nombre de booleans pour la configuration des
droits SELinux de PostgreSQL, le plus important concerne `rsync`,
souvent nécessaire pour faire des base backups. Il s'agit de
`postgresql_can_rsync`, pour l'activer :

    semanage boolean -m --on postgresql_can_rsync

Si on lance l'instance sur un port TCP différent de 5432, il faut
l'autoriser dans la configuration locale de SELinux :

    semanage port -a -t postgresql_port_t -p tcp <port>

Enfin, il ne faut pas oublier d'appliquer les contexts aux fichiers
soit avec `restorecon`, un relabel complet au reboot ou `chcon`.

[github]: http://github.com/dalibo/selinux-pgsql-pgdg
