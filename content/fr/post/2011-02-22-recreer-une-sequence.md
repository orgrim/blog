---
title: Recréer une séquence
date: "2011-02-22"
categories:
- PostgreSQL
tags:
- postgresql
- sequence
---

On peut faire ça dans 2 cas :

-   On a fait n'importe quoi, et la séquence a disparu :-(
-   On veut transformer une colonne en « SERIAL »

<!--more-->

```sql
SET ROLE owner_de_la_table;
SELECT max(id)+1 FROM latable;
--  ?column?
-- ----------
--       155
-- (1 row)

-- On prend donc « max(id)+1 » comme valeur de départ de la séquence
BEGIN;
CREATE SEQUENCE latable_id_seq START 155 OWNED BY latable.id;
COMMIT;
```

Pour lier la séquence à la table (et ainsi l'utiliser lors d'INSERT par
exemple) :

```sql
BEGIN;
ALTER TABLE schema.latable ALTER COLUMN id SET DEFAULT NEXTVAL('latable_id_seq'::regclass);
COMMIT;
```

