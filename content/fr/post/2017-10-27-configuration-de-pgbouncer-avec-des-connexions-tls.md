---
title: Configuration de pgBouncer avec des connexions TLS
date: 2017-10-27
categories:
- PostgreSQL
tags:
- pgbouncer
- auth
---

Un petit howto pour créer des certificats et les utiliser avec PostgreSQL et pgBouncer.

<!--more-->

## Autorité de certification avec EasyRSA

### Installation


Télécharger et installer la dernière version de EasyRSA. La version 3
est différente de la version 2, fournie dans Debain Stretch.

    wget https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.3/EasyRSA-3.0.3.tgz
    tar xzf EasyRSA-3.0.3.tgz
    cd EasyRSA-3.0.3

### Création de l'autorité de certification


Créer un fichier de configuration :

    cp vars.example vars
    vi vars

Changer quelques informations :

    set_var EASYRSA_REQ_COUNTRY	"FR"
    set_var EASYRSA_REQ_PROVINCE	"Ile-de-France"
    set_var EASYRSA_REQ_CITY	"Paris"
    set_var EASYRSA_REQ_ORG		"MyOrg"
    set_var EASYRSA_REQ_EMAIL	"admin@myorg.lo"
    set_var EASYRSA_REQ_OU		"Clients"

Créer l'autorité :

    ./easyrsa init-pki
    ./easyrsa build-ca

Il faut lui donner un nom, par exemple CA Clients ou PostgreSQL ou un
truc significatif. Il faut aussi saisir un mot de passe pour la clé du
certificat racine, celui qui permet de signer tout le reste.

### Création des certificats

Pour créer un bi-clé pour un serveur :

    ./easyrsa build-server-full postgresql nopass

Si la clé et le certificat du serveur doivent être transmis pas un moyen
peu sécurisé, comme le mail, créer un fichier PKCS#12 plutot qu'un tar :

    ./easyrsa export-p12 postgresql

On peut mettre un mot de passe au fichier.

    ./easyrsa build-server-full pgbouncer nopass

Générer aussi la liste de révocation, qui permet rendre des
certificats invalides.

    ./easyrsa gen-crl


### Extraction des éléments du p12


La clé privée :

    openssl pkcs12 -in postgresql.p12 -nocerts -nodes -out postgresql.key
    chmod 600 postgresql.key

Le certificat :

    openssl pkcs12 -in postgresql.p12 -clcerts -out postgresql.crt

Le certificat de l'AC :

    openssl pkcs12 -in postgresql.p12 -cacerts -out ca.crt


## Configurer PostgreSQL pour le TLS


il faut les fichiers suivants :

    pki/ca.crt
    pki/crl.pem
    pki/issued/postgresql.crt
    pki/private/postgresql.key

ou

    pki/crl.pem
    pki/private/postgresql.p12

et extraire le contenu du fichier PKCS#12 (voir plus haut).

Le plus simple est de placer ces fichiers dans le répertoire $PGDATA,
les chemins dans la configuration sont relatifs à ce répertoire.

Dans $PGDATA/postgresql.conf, configurer au minimum :

    ssl = on
    ssl_cert_file = 'postgresql.crt'
    ssl_key_file = 'postgresql.key'
    ssl_ca_file = 'ca.crt'
    ssl_crl_file = 'crl.pem'

Un reload de la configuration suffit pour prendre en compte les
modifications. Ce paramétrage permet à l'instance de chiffrer les
connexions. On définit si la connexion TLS est imposé au clients et
l'authentification par certificat dans le fichier pg_hba.conf

Le TLS ne concerne pas les connections locales par socket unix, géré
par le type 'local'.

Le type 'host', concerne les connexions en clair ou TLS, utiliser
'hostssl' ou 'hostnossl' pour distinguer le TLS dans les
autorisations.

On peut interdire les connexions en clair avec ces deux lignes :

    hostnossl   all   all   0.0.0.0/0   reject
    hostnossl   all   all   ::/0        reject

Toutes les lignes 'host' suivantes ne concerneront que les connexions
TLS, du fait de l'interdiction précédente.

## Configurer pgBouncer pour le TLS


### Clients


il faut les fichiers suivants :

    pki/ca.crt
    pki/crl.pem
    pki/issued/postgresql.crt
    pki/private/postgresql.key

ou

    pki/crl.pem
    pki/private/postgresql.p12

et extraire le contenu du fichier PKCS#12 (voir plus haut).

Dans /etc/pgbouncer/pgbouncer.ini, il faut configurer 'client_tls_sslmode' :

* disable : n'accepte pas les connexions TLS, le paramétrage par défaut
* allow : accepte les connexions TLS et les connexions en clair
* require : n'accepte que les connexions TLS
* verify-ca, verify-full : n'accepte que les connexions TLS, et demande un certificat au client issue de la même AC

Pour les chemins vers les fichiers :

    client_tls_ca_file = /etc/pgbouncer/ca.crt
    client_tls_key_file = /etc/pgbouncer/pgbouncer.key
    client_tls_cert_file = /etc/pgbouncer/pgbouncer.crt

Pour authentifier les clients avec un certificat, il faut :

    auth_type = cert

Le CN du certificat doit alors être identique au nom d'utilisateur de
connexion.

Cela ne dispense pas de la configuration de l'authenfication entre
pgBouncer et PostgreSQL, avec 'auth_file', voire 'auth_user' et 'auth_query'.

### Backends PostgreSQL


Pour que pgBouncer se connecte aux serveurs PostgreSQL en TLS, dans
/etc/pgbouncer/pgbouncer.ini, il faut configurer :

    server_tls_sslmode = require

Si on souhaite que pgBouncer vérifie le certificat de l'AC du serveur
PostgreSQL :

    server_tls_sslmode = verify-ca
    server_tls_ca_file = /etc/pgbouncer/ca.crt

On peut enfin configurer pgBouncer et PostgreSQL pour que pgBouncer
s'authentifie avec un certificat client, dans /etc/pgbouncer.ini :

    server_tls_sslmode = verify-ca
    server_tls_ca_file = /etc/pgbouncer/ca.crt
    server_tls_key_file = /etc/pgbouncer/pgbouncer_back.key
    server_tls_cert_file = /etc/pgbouncer/pgbouncer_back.crt

Le fichier pg_hba.conf devra alors demander un certificat avec
l'option clientcert=1 :

    hostssl all all ip_pgbouncer/32 md5 clientcert=1


On ne peut utiliser la méthode cert uniquement si le CN du certificat
client de pgBouncer est identique au nom de l'utilisateur qui se
connecte. Ainsi, on ne peut plus utiliser qu'un seul nom d'utilisateur
pour les accès à PostgreSQL, dans ce cas là il vaut mieux forcer le
nom d'utilisateur dans la section [database].

