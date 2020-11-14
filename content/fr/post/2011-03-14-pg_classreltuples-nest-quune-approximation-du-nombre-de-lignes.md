---
title: pg_class.reltuples n'est qu'une approximation du nombre de lignes
date: "2011-03-14"
categories:
- PostgreSQL
tags:
- postgresql
- pg_catalog
---

Dans le catalogue de PostgreSQL, qui donne plein d'information sur le
cluster et la base de données, la table `pg_class` regroupe les
informations sur les tables, les séquences, les index, tout ce qui
contient des colonnes, appelé relation. L'un des champs disponibles est
`reltuples`, il contient le nombre approximatif de lignes dans une
relation. C'est *approximatif*, ainsi ce n'est pas un donnée sure... La
preuve par l'exemple.

<!--more-->

On crée un table avec une colonne de type entier :

    
    db=# CREATE TABLE truc (i int);
    CREATE TABLE
    

On retrouve ensuite notre table ainsi que la valeur de `reltuples` dans
le catalogue système :

    
    db=# SELECT relname, reltuples FROM pg_class WHERE relname = 'truc';
     relname | reltuples 
    ---------+-----------
     truc    |         0
    (1 row)
    
    

La table est vide, et `reltuples` est à zéro. On insère ensuite quelques
lignes dans cette table :

    
    db=# INSERT INTO dalibo.truc VALUES (1),(2),(3),(4);
    INSERT 0 4
    

On regarde la valeur de `reltuples` pour la table après cette
insertion :

    
    db=# SELECT relname, reltuples FROM pg_class WHERE relname = 'truc';
     relname | reltuples 
    ---------+-----------
     truc    |         0
    (1 row)
    
    

`reltuples` n'est pas à jour, il vaut toujours zéro alors que la table
contient 4 lignes !

Comme indiqué dans la documentation, `reltuples` est mis à jour lors
d'un `ANALYSE` ou d'autres opérations de type DDL :

    
    db=# ANALYZE truc;
    ANALYZE
    db=# SELECT relname,reltuples FROM pg_class WHERE relname = 'truc';
     relname | reltuples 
    ---------+-----------
     truc    |         4
    (1 row)
    
    

Et voilà, le passage du `ANALYSE` a permis de mettre à jour la valeur de
`reltuples` pour la table.

Conclusion, si on désire supprimer une table parce qu'elle est vide, le
moyen sûr est de compter le nombre de lignes dans la table et non pas se
baser sur la valeur de `reltuples`.
