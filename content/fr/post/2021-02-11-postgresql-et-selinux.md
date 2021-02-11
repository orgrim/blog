---
title: "PostgreSQL et SELinux à l'ère de systemd"
date: 2021-02-11T10:00:00+01:00
categories:
- PostgreSQL
tags:
- postgresql
- selinux
---

Ce post se veut une mise à jour du [post précédent]({{< relref
"2014-09-22-confiner-postgresql-avec-selinux" >}}) sur le sujet. En
effet, le module SELinux, les commandes utilisées ont évolué avec
l'adoption de systemd. Les principes restent les mêmes.

<!--more-->

Pour confiner une instance PostgreSQL avec SELinux, il faut configurer
plusieurs éléments :

* Les « file contexts » : les binaires et les données doivent avoir le
  bon type dans la famille des contexts PostgreSQL. Cela se fait avec
  des regexp et la commande `semanage fcontext` en local ou par un
  module SELinux.
  
  Par exemple pour créer une instance dans `/srv/pgsql` :

  ``` console
  # semanage fcontext -a -t postgresql_db_t /srv/pgsql(/.*)?
  ```
  
  On ajoutera des règles similaires pour les tablespaces.

* le port TCP : dans la politique par défaut du système le port 5432
  est bien taggué, pour faire écouter PostgreSQL sur un autre port, il
  ne doit pas être taggué avec un autre type SElinux et explicitement
  taggué avec le type `postgresql_port_t` avec la commande `semanage
  port`.
  
  Par exemple pour faire écouter l'instance sur le port 5433 :
  
  ``` console
  # semanage port -a -p tcp -t postgresql_port_t 5433
  ```

* activer des « boolean » SELinux : par exemple pour autoriser
  l'utilisation de `rsync` par un processus PostgreSQL confiné, utile
  pour l'archivage des WAL.
  
  Par exemple :
  
  ``` console
  # semanage boolean -m --on postgresql_can_rsync
  ```

Le module `postgresql-pgdg` disponible ici :
<https://github.com/dalibo/selinux-pgsql-pgdg> fournit des file
contexts et boolean pour cela.

Ensuite, il y a deux moyens pour confiner *vraiment* l'instance :

* Lancer le postmaster avec systemd
* Lancer `pg_ctl` avec `runcon`

Avec systemd, c'est automatique, mais il faut être root ou avoir
l'autorisation de le faire avec sudo.

Avec `pg_ctl`, c'est un peu plus délicat. Avec la politique par
défaut, on a le plus souvent un shell de l'utilisateur `postgres` non
confiné, donc avec l'user SELinux `unconfiend_u`. Cet user SELinux a
les roles `system_r` et `unconfined_r`, il peut donc exécuter des
process avec le role `system_r` (voir la sortie `semanage user
-l`). Cela permet d'utiliser `runcon` pour exécuter `pg_ctl` dans le
même context que celui utilisé par systemd pour confiner PostgreSQL :

``` console
$ runcon system_u:system_r:postgresql_t:s0 pg_ctl -D /srv/pgsql/12/main start
$ ps xfZ
LABEL                             PID TTY      STAT   TIME COMMAND
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 1317 pts/0 S   0:00 -bash
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 1334 pts/0 R+   0:00  \_ ps xfZ
system_u:system_r:postgresql_t:s0 1144 ?       Ss     0:00 /usr/pgsql-12/bin/postgres -D /srv/pgsql/12/main
system_u:system_r:postgresql_t:s0 1176 ?       Ss     0:00  \_ postgres: main: logger
system_u:system_r:postgresql_t:s0 1178 ?       Ss     0:00  \_ postgres: main: checkpointer
system_u:system_r:postgresql_t:s0 1179 ?       Ss     0:00  \_ postgres: main: background writer
system_u:system_r:postgresql_t:s0 1180 ?       Ss     0:00  \_ postgres: main: walwriter
system_u:system_r:postgresql_t:s0 1181 ?       Ss     0:00  \_ postgres: main: autovacuum launcher
system_u:system_r:postgresql_t:s0 1182 ?       Ss     0:00  \_ postgres: main: archiver   last was 00000001000000000
system_u:system_r:postgresql_t:s0 1183 ?       Ss     0:00  \_ postgres: main: stats collector
system_u:system_r:postgresql_t:s0 1184 ?       Ss     0:00  \_ postgres: main: logical replication launcher
```

Enfin, la sauvegarde, du fait que l'archivage soit exécuté par
l'instance, est impactée par SELinux. En plus des droits unix
classiques, le context `postgresql_t` doit pouvoir écrire dans le
répertoire d'archivage s'il est local. Il faut donc que ce répertoire
ait le contexte `postgresql_db_t`. Ce n'est pas forcément suffisant si
le répertoire de stockage des WAL archivés est dans un point de
montage réseau. Il existe pour cela les booléens :

* `postgresql_pgdg_use_nfs` pour les montages NFS
* `postgresql_pgdg_use_fusefs` pour les montages plus exotiques utilisant FUSE

Le paquet RPM pour RHEL/CentOS est disponible dans le dépot
<https://yum.dalibo.org/labs/>, sous le nom
`selinux-policy-pgsql-pgdg`.

