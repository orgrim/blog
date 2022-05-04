---
title: "SET SESSION AUTHORIZATION vs SET ROLE"
date: 2022-05-04T09:00:00+02:00
categories:
- PostgreSQL
tags:
- postgresql
---

Sur PostgreSQL, pour obtenir les permissions et attributs d'un autre rôle, on
peut utiliser `SET SESSION AUTHORIZATION` ou `SET ROLE`. Si on est autorisé à le
faire, le résultat semble être le même : on se fait passer pour le rôle, mais il
y a quelques subtilités.

<!--more-->

Déjà, `SET SESSION AUTHORIZATION` est réservé aux super-utilisateurs. Les rôles
non super-utilisateurs ne peuvent changer de rôle qu'avec `SET ROLE` vers un
rôle dont ils sont membres.

Après un `SET SESSION AUTHORIZATION`, on a changé de rôle de session :
on ne peut plus faire `SET ROLE` que vers un rôle dont le rôle de session courant
est membre.

Ainsi, `SET SESSION AUTHORIZATION` permet de réellement se mettre à la place
d'un rôle non-privilégié.

Pour illustrer, créons un certain nomdre de rôles dans une instance fraîchement
créée :

```
create role adm noinherit;
create role alice login;
create role bob login;
grant adm to alice;
```

Connecté en temps que super-utilisteur de l'instance, on a les rôles suivants :

```
postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 adm       | No inheritance, Cannot login                               | {}
 alice     |                                                            | {adm}
 bob       |                                                            | {}
 orgrim    | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 super     | Create DB, Cannot login                                    | {}

```

Il y a plusieurs « niveaux » de rôle dans PostgreSQL, pour implémenter tout ça :

* le rôle authentifié, c'est celui fournit par le client à la connexion. Il ne
  change pas durant la vie de la session.
* le rôle de session, identique au rôle authentifié, il peut être changé avec
  `SET SESSION AUTHORIZATION`.
* le rôle courant, qui peut être modifié par SET ROLE ou le contexte d'exécution
  (fonctions notamment).

Côté code, il y en a d'autres, on se limite à ces trois là car ils sont visibles.

La requête suivante donne l'information, juste après le login en temps que
super-utilisateur `orgrim` :

```
postgres=# select session_user, current_user;
 session_user | current_user 
--------------+--------------
 orgrim       | orgrim
(1 row)
```

Avec `SET ROLE`, on peut devenir tous les rôles accessibles à `session_user` :

```
postgres=# select session_user, current_user;
 session_user | current_user 
--------------+--------------
 orgrim       | orgrim
(1 row)

postgres=# set role alice;
SET
postgres=> select session_user, current_user;
 session_user | current_user 
--------------+--------------
 orgrim       | alice
(1 row)

postgres=> set role bob;
SET
postgres=> select session_user, current_user;
 session_user | current_user 
--------------+--------------
 orgrim       | bob
(1 row)
```

`SET SESSION AUTHORIZATION` permet de changer `session_user`, il devient aussi
`current_user` :

```
postgres=# set session authorization alice;
SET
Time: 0.214 ms
postgres=> select session_user, current_user;
 session_user | current_user 
--------------+--------------
 alice        | alice
(1 row)
```

Désormais on ne peut changer de rôle que pour un rôle dont `alice` est membre :

```
postgres=> set session authorization alice;
SET
postgres=> select session_user, current_user;
 session_user | current_user 
--------------+--------------
 alice        | alice
(1 row)

postgres=> set role bob;
ERROR:  permission denied to set role "bob"
postgres=> set role adm;
SET
postgres=> select session_user, current_user;
 session_user | current_user 
--------------+--------------
 alice        | adm
(1 row)
```

Dans ce dernier exemple, on a vraiment incarné `alice` comme si elle s'était
connectée.

PostgreSQL, grâce au rôle authentifié peut annuler le changement, avec `RESET
SESSION AUTHORIZATION`.

On pourrait penser l'intérêt de `SET SESSION AUTHORIZATION` limité du fait
qu'il faille être super-utilisateur pour l'utiliser. C'est juste très utile en
temps que super-utilisateur, par exemple pour pouvoir valider qu'un script DDL
fonctionne dans le contexte d'un rôle non privilégier sans avoir besoin de
configurer l'authentification pour ce rôle. On peut aussi valider des
permissions et des changements de rôles dans le contexte réel d'un rôle donné.
