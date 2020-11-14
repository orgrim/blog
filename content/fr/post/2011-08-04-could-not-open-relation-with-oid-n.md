---
title: Could not open relation with oid N
date: "2011-08-04"
categories:
- PostgreSQL
tags:
- postgresql
- lock
- pg_catalog
---

On peut parfois trouver cet étrange message d'erreur dans les traces de
PostgreSQL (N étant un nombre) ou lors de l'exécution d'une requête :

    
    ERROR:  could not open relation with OID N
    

Si on recherche ce message dans les mailing-lists du projet, on peut
facilement conclure que la base de données est corrompue, qu'il y a des
problèmes matériels et que la sécurité des données est en péril. Et
bien, ce n'est pas forcément le cas : obtenir ce message peut être tout
à fait normal.

Pour démontrer cela, on a besoin d'une table :

    
    $ createdb test
    $ psql test
    psql (9.0.4)
    Type "help" for help.
    
    test=# CREATE TABLE truc AS SELECT generate_series(0, 5) AS i;
    SELECT 6
    test=#
    

On lance une session qui bloquerait un `DROP` de cette table, pour cela
on pose un verrou exclusif, le mode « ExclusiveLock » ne laisse passer
que les lectures (c'est important pour la suite) :

    
    $ psql test
    psql (9.0.4)
    Type "help" for help.
    
    test=# BEGIN;
    BEGIN
    test=# LOCK TABLE truc IN EXCLUSIVE MODE;
    LOCK TABLE
    test=#
    

On laisse cette transaction « ouverte », avec le verrou posé et on lance
une session pour supprimer la table :

    
    $ psql test
    psql (9.0.4)
    Type "help" for help.
    
    test=# BEGIN;
    BEGIN
    test=# DROP TABLE truc;
    

L'ordre SQL `DROP TABLE` ne rend pas la main, cette deuxième session
attend le verrou « AccessExclusiveLock », qui est le plus restrictif,
sur la table pour pouvoir la supprimer. La page
[http://wiki.postgresql.org/wiki/Lock_dependency_information] fournie une requête montrant
les dépendances entre requêtes du point de vue du verrouillage. Dans ce
cas, elle donne le résultat suivant :

    
     waiting_locktype | waiting_table |  waiting_query   |    waiting_mode     | waiting_pid | other_locktype | other_table |      other_query      |  other_mode   | other_pid | other_granted 
    ------------------+---------------+------------------+---------------------+-------------+----------------+-------------+-----------------------+---------------+-----------+---------------
     relation         | truc          | DROP TABLE truc; | AccessExclusiveLock |       25632 | relation       | truc        | <IDLE> in transaction | ExclusiveLock |     24217 | t
    

On lance une troisième session, avec un `SELECT` sur notre table :

    
    $ psql test
    psql (9.0.4)
    Type "help" for help.
    
    test=# SELECT * FROM truc;
    

L'ordre `SELECT` ne rend pas la main, cette troisième session se met à
attendre le `DROP TABLE` et la première session. C'est d'ailleurs le
`DROP TABLE` qui bloque réellement le `SELECT`, car la première session
à verrouillé la table en lecture seule :

    
     waiting_locktype | waiting_table |    waiting_query    |    waiting_mode     | waiting_pid | other_locktype | other_table |      other_query      |     other_mode      | other_pid | other_granted 
    ------------------+---------------+---------------------+---------------------+-------------+----------------+-------------+-----------------------+---------------------+-----------+---------------
     relation         | truc          | SELECT * FROM truc; | AccessShareLock     |       28629 | relation       | truc        | DROP TABLE truc;      | AccessExclusiveLock |     25632 | f
     relation         | truc          | DROP TABLE truc;    | AccessExclusiveLock |       25632 | relation       | truc        | <IDLE> in transaction | ExclusiveLock       |     24217 | t
     relation         | truc          | SELECT * FROM truc; | AccessShareLock     |       28629 | relation       | truc        | <IDLE> in transaction | ExclusiveLock       |     24217 | t
     relation         | truc          | DROP TABLE truc;    | AccessExclusiveLock |       25632 | relation       | truc        | SELECT * FROM truc;   | AccessShareLock     |     28629 | f
    (4 rows)
    

On libère la première session :

    
    test=# ROLLBACK;
    ROLLBACK
    test=#
    

Le `DROP TABLE` passe, et le `SELECT` continue d'attendre :

    
     waiting_locktype | waiting_table |    waiting_query    |  waiting_mode   | waiting_pid | other_locktype | other_table |      other_query      |     other_mode      | other_pid | other_granted 
    ------------------+---------------+---------------------+-----------------+-------------+----------------+-------------+-----------------------+---------------------+-----------+---------------
     relation         | truc          | SELECT * FROM truc; | AccessShareLock |       28629 | relation       | truc        | <IDLE> in transaction | AccessExclusiveLock |     25632 | t
    

On voit que le `SELECT` attend la transaction qui a lancé le
`DROP TABLE`. Même si le `DROP TABLE` est terminé, son effet ne sera
connu des transactions concurrentes seulement ou moment du commit ou
rollback, parce qu'on utilise le niveau d'isolation des transaction
« read committed » (par défaut). Il n'y a pas de « `UNLOCK` » sur les
tables dans PostgreSQL, il faut attendre la fin de la transaction pour
que les verrous soient libérés, du moins lorsqu'on n'utilise pas de
savepoints.

Maintenant, on valide le `DROP TABLE`, avec l'ordre `COMMIT`. Le
`SELECT` termine en erreur, on voit alors le message dans les logs :

    
    ERROR:  could not open relation with OID 17366 at character 15
    STATEMENT:  SELECT * FROM truc;
    

Lorsque le `SELECT` n'est plus bloqué par le verrou, il ne peut accéder
pas à la table car elle n'existe plus. Le message n'est pas très
explicite parce que la requête est en cours d'exécution : le moteur a
déjà terminé le travail de parsing et de planification, il ne travaille
qu'avec les OID qu'il a récupéré du catalogue système à ce moment là.

Dans ce cas précis, obtenir ce message n'est un problème de corruption
de la base ou du catalogue système.

</p>

[http://wiki.postgresql.org/wiki/Lock_dependency_information]: http://wiki.postgresql.org/wiki/Lock_dependency_information
