---
title: Gestion des utilisateurs pgBouncer avec auth_query
date: 2017-10-30
categories:
- PostgreSQL
tags:
- pgbouncer
- auth
---

Avec pgBouncer 1.6 sont arrivés deux nouveaux paramètres, `auth_user` et
`auth_query`, qui permettent de faciliter la gestion du fichier des
utilisateurs. Le fichier des utilisateurs contient les couples
utilisateur / mot de passe hashé pour s'authentifier à la place de
l'utilisateur final sur le serveur PostgreSQL. Il y a deux problèmes
majeurs avec ce fichier utilisateur : son maintien fastidieux et le
risque de collision entre noms de roles identiques mais avec des mots
de passe différents si le même pgBoucner accède à plusieurs instances
PostgreSQL.

<!--more-->

`auth_query` permet de définir une requête à exécuter pour obtenir le
couple utilisateur / mot de passe de l'instance lorsqu'ils sont
introuvables dans le fichier.

Pour se connecter à l'instance PostgreSQL et exécuter la requête,
pgBouncer utilise la valeur de `auth_user` comme nom d'utilisateur. Si
PostgreSQL demande un mot de passe, pgBouncer utilise celui défini
dans le fichier des utilisateurs. On peut utiliser un utilisateur
commun à toutes les instances, qu'il faut alors créer sur chaque
instance, ou ajouter le paramètre `auth_user` au niveau de la définir
de l'accès à la base de données dans la section [databases].

Enfin, l'utilisateur `auth_user` doit pouvoir lire le contenu de
`pg_shadow` pour obtenir les mots de passe hashés, il est alors
recommandé de créer une fonction dans chaque base de données accédée
par pgBouncer sur l'instance cible. Sinon, il faut donner accès à
`pg_shadow` par un `GRANT`.

Concrètement cela donne, dans `/etc/pgbouncer/pgbouncer.ini` :

```ini
[pgbouncer]

; ...

auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

;; si on choisit de faire un GRANT sur pg_shadow
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1

;; si on choisit de créer la fonction donnée dans la doc de pgBouncer
auth_query = SELECT pgbouncer.user_lookup($1)
```

Sur l'instance PostgreSQL, pour créer l'utilisateur pgbouncer chargé
d'exécuter l'auth_query :

```sql
CREATE ROLE pgbouncer WITH PASSWORD 'le mot de passe';
SELECT usename, passwd FROM pg_catalog.pg_shadow WHERE username = 'pgbouncer';
```

Créer le fichier `/etc/pgbouncer/userlist.txt` et mettre le résultat du SELECT sous cette forme :

    "pgbouncer" "md5ac0b6e3b0d0904c0b602607e36c256eb"

Si on choisit de faire un GRANT sur `pg_shadow`, faire cette opération
dans toutes les bases de données accédées via pgBouncer :

```sql
GRANT SELECT ON pg_catalog.pg_shadow TO pgbouncer;
```

Sinon exécuter le script SQL suivant dans toutes les bases de données
accédées via pgBouncer, c'est la fonction donnée dans la doc avec la
création du schéma en plus :

```sql
CREATE SCHEMA pgbouncer;
REVOKE ALL ON SCHEMA pgbouncer FROM public, pgbouncer;
GRANT USAGE ON SCHEMA pgbouncer TO pgbouncer;

CREATE OR REPLACE FUNCTION pgbouncer.user_lookup(in i_username text, out uname text, out phash text)
RETURNS record AS $$
BEGIN
    SELECT usename, passwd FROM pg_catalog.pg_shadow
    WHERE usename = i_username INTO uname, phash;
    RETURN;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
REVOKE ALL ON FUNCTION pgbouncer.user_lookup(text) FROM public, pgbouncer;
GRANT EXECUTE ON FUNCTION pgbouncer.user_lookup(text) TO pgbouncer;
```

On peut ajouter la fonction ou le `GRANT` à `template1` pour les futures bases de données.
