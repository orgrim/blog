---
title: Oups... des tablespaces imbriqués
date: "2013-04-06"
categories:
- PostgreSQL
tags:
- postgresql
- pg_catalog
---

Imbriquer des tablespaces n'a pas vraiment de sens dans PostgreSQL
surtout si on veut se prendre la tête avec des montages dans tous les
sens... Mais bon c'est permis, car PostgreSQL utilise uniquement les
liens symboliques dans `$PGDATA/pg_tblspc` pour accéder au contenu des
tablespaces.

<!--more-->

Si on veut savoir ce qu'il en est voici une requête qui montre ça.

Pour PostgreSQL \<= 9.1 :

```sql
SELECT f.oid, f.spcname AS name, f.spclocation AS location,
  f.spclocation ~ ('^' || (SELECT setting FROM pg_settings
                           WHERE name = 'data_directory')) AS inside_pgdata,
  (SELECT o.spcname
   FROM pg_tablespace o
   WHERE f.spclocation ~ ('^' || o.spclocation || '/')
     AND o.spcname NOT IN ('pg_default', 'pg_global')
   ORDER BY o.spclocation DESC LIMIT 1) AS parent,
  (SELECT o.spclocation   FROM pg_tablespace o
   WHERE f.spclocation ~ ('^' || o.spclocation || '/')
     AND o.spcname NOT IN ('pg_default', 'pg_global')
   ORDER BY o.spclocation DESC LIMIT 1) AS parent_location
FROM pg_tablespace f
WHERE f.spcname NOT IN ('pg_default', 'pg_global');
```

Pour PostgreSQL \>= 9.2, la colonne `spcname` de `pg_tablespace` a été
remplacée par la fonction `pg_tablespace_location()` :

```sql
SELECT f.oid, f.spcname AS name, pg_tablespace_location(f.oid) AS location,
  pg_tablespace_location(f.oid) ~ ('^' || (SELECT setting FROM pg_settings
                                    WHERE name = 'data_directory')) AS inside_pgdata,
  (SELECT o.spcname
   FROM pg_tablespace o
   WHERE pg_tablespace_location(f.oid) ~ ('^' || pg_tablespace_location(o.oid) || '/')
     AND o.spcname NOT IN ('pg_default', 'pg_global')
   ORDER BY pg_tablespace_location(o.oid) DESC LIMIT 1) AS parent,
  (SELECT pg_tablespace_location(o.oid)
   FROM pg_tablespace o
   WHERE pg_tablespace_location(f.oid) ~ ('^' || pg_tablespace_location(o.oid) || '/')
     AND o.spcname NOT IN ('pg_default', 'pg_global')
   ORDER BY pg_tablespace_location(o.oid) DESC LIMIT 1) AS parent_location
FROM pg_tablespace f
WHERE f.spcname NOT IN ('pg_default', 'pg_global');
```

On obtient par exemple :

    
      oid  |   name   |                       location                       | inside_pgdata | parent  |                 parent_location                 
    -------+----------+------------------------------------------------------+---------------+---------+-------------------------------------------------
     16399 | inside   | /home/pgsql/postgresql-9.0.10/data/inside            | t             |         | 
     16400 | outside  | /home/pgsql/postgresql-9.0.10/outside                | f             |         | 
     16414 | imbrique | /home/pgsql/postgresql-9.0.10/outside/imbrique       | f             | outside | /home/pgsql/postgresql-9.0.10/outside
     16415 | rimb     | /home/pgsql/postgresql-9.0.10/outside/rimb           | f             | outside | /home/pgsql/postgresql-9.0.10/outside
     16416 | rimbimb  | /home/pgsql/postgresql-9.0.10/outside/rimb/imb       | f             | rimb    | /home/pgsql/postgresql-9.0.10/outside/rimb
     16417 | deuimb   | /home/pgsql/postgresql-9.0.10/outside/rimb/truc/2imb | f             | truc    | /home/pgsql/postgresql-9.0.10/outside/rimb/truc
     16418 | truc     | /home/pgsql/postgresql-9.0.10/outside/rimb/truc      | f             | rimb    | /home/pgsql/postgresql-9.0.10/outside/rimb
    (7 rows)
    

Pour bien gérer ses tablespaces, l'idéal est de faire un répertoire
dédié au même niveau que `$PGDATA`, et de créer les tablespaces dedans,
avec un tablespace par répertoire sur un seul niveau :

    
     /home/pgsql/postgresql-9.2.4
    ├── data
    └── tblspc
        ├── tbs_1
        ├── tbs_2
        ├── ...
        └── tbs_n
    

C'est plus propre et on voit tout de suite l'utilisation disque avec
`df` : parce qu'on met des disques différents derrière, bien entendu...
:)
