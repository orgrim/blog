---
title: Réflexions sur l'archivage des fichiers WAL
date: 2015-12-30
categories:
- PostgreSQL
tags:
- archiving
- wal
- replication
---

PostgreSQL faisant son bonhomme de chemin, on se retrouve désormais
avec des configurations où il faut archiver plusieurs fois les WAL
parce qu'on a de la sauvegarde PITR et de la réplication.

<!--more-->

Le plus gros piège lorsqu'on a du PITR et de la réplication, ou
plusieurs serveurs standby, c'est d'oublier que chaque élément de
l'architecture qui consomme du WAL doit avoir son propre répertoire de
WAL archivés, car chacun purge les fichiers différemment.

On tombe dans le piège facilement, en se disant, "pas de problème pour
la purge des vieux fichiers, c'est le PITR ou le slave le plus éloigné
qui purgera". Si la purge du PITR passe alors que le standby était
déconnecté du maitre, cette purge peut casser la réplication.

La solution est d'archiver plusieurs fois le même fichier WAL. Pour
optimiser, le plus efficace à l'usage est d'utiliser des
hardlinks. En gros, on archive une fois le fichier WAL et on crée
autant de lien hard qu'il faut pour les autres consommateurs de WAL
archivés. Rappelons, que la donnée n'est supprimée que lorsqu'il
n'existe plus aucun lien et que plus aucun processus n'a le fichier
ouvert, à ne pas confondre avec un lien symbolique.

Pour archiver vite, il vaut mieux éviter de compresser et stocker les
archives soit en local, soit sur un partage NFS, l'archivage par SSH
restant le plus lent. Tout est compromis entre chiffrage des
communications sur le réseau et espace disque disponible, les liens
hard restant rapides à créer et avec une consommation d'espace disque
supplémentaire négligeable.

Enfin, PostgreSQL exécute la commande d'archivage avec l'appel
`system()` qui fork un shell : toutes les possibilités du shell sont
alors disponibles, par exemple :

    archive_command = 'rsync -a %p slave1:/archived_xlog/slave1/%f && ssh slave1 "for h in slave2 pitr; do ln /archived_xlog/slave1/%f /archived_xlog/$h/%f; done"'

Oui, une boucle et une seule copie avec rsync par SSH pour 3
utilisations. On préfèrera surement faire un script pour rendre les
choses plus lisibles. Ça marche aussi pour `restore_command` et
`archive_cleanup_command` dans `recovery.conf` :

    restore_command = 'cp /archived_xlog/$(hostname)/%f %p'
    archive_cleanup_command = 'pg_archivecleanup /archived_xlog/$(hostname) %r'

