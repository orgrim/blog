---
title: Retrouver un trigger à partir du nom de la fonction associée
date: "2011-04-07"
categories:
- PostgreSQL
tags:
- postgresql
- pg_catalog
---

Ici encore, tout est dans le catalogue système de PostgreSQL, il suffit
de regarder dans la table `pg_trigger`.

<!--more-->

Avec les jointures qui vont bien, on peut obtenir les informations
intéressantes :

```sql
SELECT nspname AS "schema", relname AS "table", tgname AS "trigger", proname AS "function"
FROM pg_trigger t
  JOIN pg_class c ON (t.tgrelid = c.oid)
  JOIN pg_namespace n ON (n.oid = c.relnamespace)
  JOIN pg_proc p ON (p.oid = t.tgfoid);
```

En ajoutant une clause WHERE, on peut facilement retrouver la table
associée au trigger :

```sql
SELECT nspname AS "schema", relname AS "table", tgname AS "trigger", proname AS "function"
FROM pg_trigger t
  JOIN pg_class c ON (t.tgrelid = c.oid)
  JOIN pg_namespace n ON (n.oid = c.relnamespace)
  JOIN pg_proc p ON (p.oid = t.tgfoid)
WHERE proname = 'mafonction';
```
