---
title: Quand PostgreSQL n'a plus d'espace disque à manger
date: "2011-10-14"
categories:
- PostgreSQL
tags:
- postgresql
---

Voilà donc une question intéressante, comment se comporte PostgreSQL
face à un système de fichier plein ? Un peu d'expérimentation est
nécessaire pour se rassurer...

On crée deux systèmes de fichiers de faible taille pour les tests. Le
premier stockera `PGDATA`, ainsi qu'une base de données nommée `db_data`. Le
second sera le tablesapce `ts1`, dans lequel oncréera une base de données
`db_ts1`.

L'objectif est de montrer que seules les transactions modifiant des
objets stockés sur des systèmes de fichier plein sont affectées, c'est
pourquoi on a besoin de plusieurs tablespaces.

Les binaires se trouvent dans `/home/pgsql/postgresql-9.0.4`, `PGDATA` dans
`/home/pgsql/postgresql-9.0.4/data`, et le tablespace dans
`/home/pgsql/postgresql-9.0.4/ts1`. On a donc monté et donné la propriété
des deux filesystèmes à orgrim, que fera tourner PostgreSQL :

    
    # df -h
    ...
    /dev/mapper/sys-pg1   504M   54M  425M  12% /home/pgsql/postgresql-9.0.4/data
    /dev/mapper/sys-pg2   504M   17M  462M   4% /home/pgsql/postgresql-9.0.4/ts1
    ...
    
    # cd /home/pgsql/postgresql-9.0.4/
    # chown orgrim: data ts1
    # chmod 700 data ts1
    

Le cluster est préparé avec l'environnement suivant :

    
    $ env | grep PG
    PGPORT=5904
    PGDATABASE=postgres
    PGDATA=/home/pgsql/postgresql-9.0.4/data
    PATH=/home/pgsql/postgresql-9.0.4/bin:$PATH
    

On lance donc `initdb`, puis on crée les bases avec le tablespace :

    
    $ initdb
    $ psql
    psql (9.0.4)
    Type "help" for help.
    
    postgres=# CREATE DATABASE db_data;
    CREATE DATABASE
    postgres=# CREATE TABLESPACE ts1 LOCATION '/home/pgsql/postgresql-9.0.4/ts1';
    CREATE TABLESPACE
    postgres=# CREATE DATABASE db_ts1 TABLESPACE ts1;
    CREATE DATABASE
    

Ensuite, on se connecte à la base de données `db_ts1` et on y crée une
base qui permettra de remplir le tablespace `ts1` :

    
    $ while ((1)); do psql -c "INSERT INTO t SELECT generate_series(1, 1000) AS i;" db_ts1; done
    

Au bout, d'un moment le système de fichier du tablespace est plein et
tout requête générant des écritures sort en erreur avec un message de
cette forme :

    
    ERROR:  could not extend file "pg_tblspc/16385/PG_9.0_201008051/16386/16390": wrote only 4096 of 8192 bytes at block 58438
    HINT:  Check free disk space.
    

Ensuite, on essaye avec le répertoire `PGDATA`, on crée donc une table
dans la base `db_data` :

    
    $ psql db_data
    psql (9.0.4)
    Type "help" for help.
    
    db_data=# CREATE TABLE t (i int);
    CREATE TABLE
    

De la même façon, on remplit cette table jusqu'à épuisement de l'espace
libre :

    
    $ while ((1)); do psql -c "INSERT INTO t SELECT generate_series(1, 1000) AS i;" db_data; done
    

Résultat, les requêtes sortent en erreur de la même façon. On a
peut-être de la chance ici, le filesystem contenant `pg_xlog`, les
problèmes pourraient être pis.

Il est également possible de remplir le système de fichiers de journaux
de transactions, ce qui est problématique du fait que chaque transaction
est écrite ici tout tablespace confondu. On vide donc les deux bases :

    
    $ psql db_ts1
    psql (9.0.4)
    Type "help" for help.
    
    db_ts1=# TRUNCATE t;
    TRUNCATE TABLE
    
    $ psql db_data
    psql (9.0.4)
    Type "help" for help.
    
    db_data=# TRUNCATE t;
    TRUNCATE TABLE
    

Et on place le paramètre `checkpoint_segments` à 300, valeur démesurée
par rapport à la place disponible dans `PGDATA`.

Après un reload, on refait alors le test de remplissage de la base
`db_ts1`, qui assure que les journaux de transactions seuls remplissent
`PGDATA`.

Le serveur PostgreSQL s'arrête parce qu'il ne peut écrire le journal de
transaction :

    
    PANIC:  could not write to file "pg_xlog/xlogtemp.8795": No space left on device
    STATEMENT:  INSERT INTO t SELECT generate_series(1, 1000) AS i;
    LOG:  server process (PID 8795) was terminated by signal 6: Aborted
    LOG:  terminating any other active server processes
    WARNING:  terminating connection because of crash of another server process
    DETAIL:  The postmaster has commanded this server process to roll back the current transaction and exit, because another server process exited abnormally and possibly corrupted shared memory.
    HINT:  In a moment you should be able to reconnect to the database and repeat your command.
    LOG:  all server processes terminated; reinitializing
    

Il redémarre illico, mais le problème persiste :

    
    FATAL:  the database system is in recovery mode
    LOG:  database system was interrupted; last known up at 2011-10-12 00:01:40 CEST
    LOG:  database system was not properly shut down; automatic recovery in progress
    LOG:  consistent recovery state reached at 0/5FDB7480
    LOG:  redo starts at 0/5AA57D68
    LOG:  could not open file "pg_xlog/000000010000000000000074" (log file 0, segment 116): No such file or directory
    LOG:  redo done at 0/73FFFF90
    LOG:  last completed transaction was at log time 2011-10-12 00:05:11.710639+02
    PANIC:  could not write to file "pg_xlog/xlogtemp.8797": No space left on device
    LOG:  startup process (PID 8797) was terminated by signal 6: Aborted
    LOG:  aborting startup due to startup process failure
    

Le seul moyen de se sortir de cette situation est d'utiliser les 5%
d'espace libre du filesystem réservés au super utilisateur, de baisser
la valeur de `checkpoint_segments` à une valeur évitant le filesystem
full, les checkpoints successif supprimeront les fichiers en trop.

Lorsqu'on a plus besoin des 5% réservés, il ne faut pas oublier de
remettre la réservation.

Enfin, ce cas le plus critique peut arriver assez facilement lorsqu'on a
de l'archivage, si le serveur ne peut plus archiver les fichiers WAL,
alors il les conserve, un filesystem full sur un espace d'archivage peut
donc être transmis au serveur...

