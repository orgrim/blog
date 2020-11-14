---
title: Voir tous les champs d'une table
date: "2011-02-25"
categories:
- PostgreSQL
tags:
- postgresql
- pg_catalog
---

Avec PostgreSQL, on peut utiliser la requête suivante pour obtenir la
taille de chacune des bases de données d'un cluster :

```sql
SELECT datname AS base,
  pg_size_pretty(pg_database_size(oid)) AS taille
FROM pg_database;
```

<!--more-->

Qui donne par exemple :

    
             base          | taille  
    -----------------------+---------
     template0             | 5273 kB
     postgres              | 5385 kB
     redmine               | 9057 kB
     roundcube             | 9153 kB
     template1             | 5369 kB
     dotclear              | 6833 kB
     dspam                 | 204 MB
     exim                  | 5649 kB
     (8 rows)
    

Mais comment est-ce possible ? on a appliqué la fonction
`pg_database_size()` sur une colonne appelée `oid`, alors que la
définition de `pg_database` est la suivante :

    
    postgres=# \\d pg_database
        Table "pg_catalog.pg_database"
        Column     |   Type    | Modifiers 
    ---------------+-----------+-----------
     datname       | name      | not null
     datdba        | oid       | not null
     encoding      | integer   | not null
     datcollate    | name      | not null
     datctype      | name      | not null
     datistemplate | boolean   | not null
     datallowconn  | boolean   | not null
     datconnlimit  | integer   | not null
     datlastsysoid | oid       | not null
     datfrozenxid  | xid       | not null
     dattablespace | oid       | not null
     datacl        | aclitem[] | 
    Indexes:
        "pg_database_datname_index" UNIQUE, btree (datname), tablespace "pg_global"
        "pg_database_oid_index" UNIQUE, btree (oid), tablespace "pg_global"
    Tablespace: "pg_global"
    
    

Il n'y a pas de colonne `oid` dans cette table ! Et pourtant... Cette
colonne est tout simplement cachée, mais elle existe bel et bien. Pour
vérifier, on recherche donc dans le catalogue système la liste des
champs de la table :

```sql
SELECT attname, typname, attnum
FROM pg_attribute a
  JOIN pg_class c ON (c.oid = a.attrelid)
  JOIN pg_type t ON (t.oid = a.atttypid)
WHERE relname = 'pg_database';
```

Qui donne :

    
        attname    | typname  | attnum 
    ---------------+----------+--------
     tableoid      | oid      |     -7
     cmax          | cid      |     -6
     xmax          | xid      |     -5
     cmin          | cid      |     -4
     xmin          | xid      |     -3
     oid           | oid      |     -2
     ctid          | tid      |     -1
     datname       | name     |      1
     datdba        | oid      |      2
     encoding      | int4     |      3
     datcollate    | name     |      4
     datctype      | name     |      5
     datistemplate | bool     |      6
     datallowconn  | bool     |      7
     datconnlimit  | int4     |      8
     datlastsysoid | oid      |      9
     datfrozenxid  | xid      |     10
     dattablespace | oid      |     11
     datacl        | _aclitem |     12
    (19 rows)
    

La colonne `attnum` de `pg_attribute` indique la position de la colonne,
on remarque ainsi qu'il y a sept colonnes avec une position négative.
Ces colonnes ne sont pas montrés par la commande `\\d` de psql, ce sont
les colonnes système de la table (plus d'information sur ce qu'elles
représentent dans la [doc](http://docs.postgresql.fr/9.0/ddl-system-columns.html))

En plus, on a encore utilisé cette colonne `oid` pour la retrouver dans
la table... Maintenant, on peut être rassuré sur le catalogue système,
ce n'est qu'un ensemble de tables, auxquelles on accède en SQL
standard...
