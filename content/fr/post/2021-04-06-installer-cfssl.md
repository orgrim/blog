---
title: "Installer CFSSL"
date: 2021-04-06T22:30:00+02:00
categories:
- Unix
tags:
- tls
- linux
- go
- install
ShowToc: true
---

CFSSL est l'outil de gestion de certificats TLS de CloudFlare, son principal
avantage est de pouvoir créer des certificats en utilisant une configuration au
format JSON bien plus facile à appréhender qu'OpenSSL.

<!--more-->

## Installation

Pour créer une autorité de certification complète, incluant la gestion
des révocations de certificats et la production de CRL, il nous faut :

* Les sources de cfssl pour obtenir le schéma de la base de données certdb
* L'outil `goose` pour créer la base
* Les binaires `cfssl` et `cfssljson` pour produire les certificats

Les outils `cfssl` et `goose` sont écrits en Go, on peut soit
installer Go et les compiler directement, soit passer par docker pour
les compiler, soit télécharger les binaires déjà compilés (goose
n'étant pas disponible en binaire ni packagé dans Debian).

Télécharger les sources de cfssl :

```
$ git clone https://github.com/cloudflare/cfssl.git
$ mkdir bin
```

Installer `goose` :

```
$ docker run --rm -v "$PWD/bin":/go/bin golang:1.15 go get bitbucket.org/liamstask/goose/cmd/goose
```

Compiler les binaires `cfssl` et `cfssljson` :

```
$ docker run --rm -v "$PWD/bin":/go/bin -v "$PWD/cfssl":/src -w /src golang:1.15 make install-cfssl install-cfssljson
```

Créer un répertoire pour l'AC :

```
$ mkdir ca
```

## Base de données

Créer la base de données, on utilise sqlite pour le développement :

```
$ cd ca
$ ../bin/goose -path ../cfssl/certdb/sqlite up
goose: migrating db environment 'development', current version: 0, target: 2
OK    001_CreateCertificates.sql
OK    002_AddMetadataToCertificates.sql
```

Pour utiliser l'AC, il nous faut plusieurs fichiers de configuration :

* Un fichier de configuration pour pouvoir accéder à la base de
  données des certificats
* Un fichier de configuration pour définir les attributs des clés et
  certificats, regroupés par profils
* Un fichier de configuration pour représenter la demande de
  certificat

Créer un fichier de configuration pour l'accès à la base de données
dans `ca/db-dev.json` :

``` json
{
    "driver": "sqlite3",
    "data_source": "ca/certstore_development.db"
}
```

## Configuration

Créer le fichier de configuration pour la signature des certificats
dans `ca/config.json`. On reprend les valeurs par défaut indiquées
dans la documentation (cfssl/doc/cmd/cfssl.txt)

``` json
{
    "signing": {
        "default": {
            "usages": [
                "signing",
                "key encipherment",
                "server auth",
                "client auth"
            ],
            "expiry": "8760h"
        }
    }
}
```

On peut créer des profils de signature pour simplifier l'utilisation
courante, voir la commande `cfssl print-defaults config`.

Créer le fichier de configuration pour la CSR de l'AC, dans `ca/ca.json` :

``` json
{
    "CN": "Orgrim CA",
    "names": [
        {
            "C": "France",
            "L": "Paris",
            "O": "Grim"
        }
    ],
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    }
}
```

## Créer l'AC

### Certificat racine

On peut alors créer le certificat et la clé de l'AC :

```
$ bin/cfssl gencert -initca ca/ca.json | bin/cfssljson -bare "ca/ca"
2021/02/10 21:20:36 [INFO] generating a new CA key and certificate from CSR
2021/02/10 21:20:36 [INFO] generate received request
2021/02/10 21:20:36 [INFO] received CSR
2021/02/10 21:20:36 [INFO] generating key: rsa-2048
2021/02/10 21:20:37 [INFO] encoded CSR
2021/02/10 21:20:37 [INFO] signed certificate with serial number 108110977658349829494221455501467538586622033487
```

La sortie de `cfssl` est au format JSON, l'outil `cfssljson` permet
d'extraire le certificat, la CSR et la clé au format PEM :

```
$ ls ca/ca*
ca/ca.csr  ca/ca.json  ca/ca-key.pem  ca/ca.pem
```

On peut aussi vérifier le certificat avec openssl :

```
$ openssl x509 -in ca/ca.pem -text -noout
```

### Certificats clients et serveurs

Désormais, on peut générer des certificats, en créant une clé et une
CSR. On créer donc le ficheir de configuration pour la CSR pour la
machine `pg1.local`, dans `pg1.local.json` :

``` json
{
    "CN": "pg1.local",
    "hosts": [
        "pg1.local",
        "10.0.0.5"
    ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
           "C": "France",
            "L": "Paris",
            "O": "Grim"
        }
    ]
}
```

Générer :

```
$ bin/cfssl genkey pg1.local.json | bin/cfssljson -bare pg1.local
2021/02/10 21:33:40 [INFO] generate received request
2021/02/10 21:33:40 [INFO] received CSR
2021/02/10 21:33:40 [INFO] generating key: ecdsa-256
2021/02/10 21:33:40 [INFO] encoded CSR
```

On obtient deux fichiers, `pg1.local.csr` et `pg1.local-key.pem`. Le
détail de la CSR peut être affiché avec openssl :

```
$ openssl req -in pg1.local.csr -text -noout
```

On peut alors générer le certificat et stocker les informations dans
la base de données :

```
$ bin/cfssl sign -ca ca/ca.pem \
    -ca-key ca/ca-key.pem \
    -config ca/config.json \
    -db-config ca/db-dev.json \
    pg1.local.csr | bin/cfssljson -bare pg1.local
```

On obtient le certificat dans `pg1.local.pem` :

```
$ openssl x509 -in pg1.local.pem -text -noout
```

### Créer ses profils de certificats

La responsabilité de l'autorité de certification est définir la validité
(expiration et révocation) ainsi que l'utilisation du certificat. L'avantage de
CFSSL est la possibilité de créer des profils de certificats.

On créé ces profils dans le fichier de configuration `ca/config.json`. La
commande `bin/cfssl print-defaults config` montre les valeurs par défaut, pour
créer un profil, il faut l'ajouter dans le dictionnaire `"signing" ->
"profiles"`, voici le défaut, qui montre le profil pour un serveur (`www`) et
pour un client (`client`) :

```
$ bin/cfssl print-defaults config
{
    "signing": {
        "default": {
            "expiry": "168h"
        },
        "profiles": {
            "www": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            }
        }
    }
}
```

Les « usages » possibles sont définis dans le code source dans
[config/config.go](https://github.com/cloudflare/cfssl/blob/master/config/config.go),
cela correspond aux définitions de la [RFC 5280](https://tools.ietf.org/html/rfc5280) :

``` go
// KeyUsage contains a mapping of string names to key usages.
var KeyUsage = map[string]x509.KeyUsage{
    "signing":            x509.KeyUsageDigitalSignature,
    "digital signature":  x509.KeyUsageDigitalSignature,
    "content commitment": x509.KeyUsageContentCommitment,
    "key encipherment":   x509.KeyUsageKeyEncipherment,
    "key agreement":      x509.KeyUsageKeyAgreement,
    "data encipherment":  x509.KeyUsageDataEncipherment,
    "cert sign":          x509.KeyUsageCertSign,
    "crl sign":           x509.KeyUsageCRLSign,
    "encipher only":      x509.KeyUsageEncipherOnly,
    "decipher only":      x509.KeyUsageDecipherOnly,
}

// ExtKeyUsage contains a mapping of string names to extended key
// usages.
var ExtKeyUsage = map[string]x509.ExtKeyUsage{
    "any":              x509.ExtKeyUsageAny,
    "server auth":      x509.ExtKeyUsageServerAuth,
    "client auth":      x509.ExtKeyUsageClientAuth,
    "code signing":     x509.ExtKeyUsageCodeSigning,
    "email protection": x509.ExtKeyUsageEmailProtection,
    "s/mime":           x509.ExtKeyUsageEmailProtection,
    "ipsec end system": x509.ExtKeyUsageIPSECEndSystem,
    "ipsec tunnel":     x509.ExtKeyUsageIPSECTunnel,
    "ipsec user":       x509.ExtKeyUsageIPSECUser,
    "timestamping":     x509.ExtKeyUsageTimeStamping,
    "ocsp signing":     x509.ExtKeyUsageOCSPSigning,
    "microsoft sgc":    x509.ExtKeyUsageMicrosoftServerGatedCrypto,
    "netscape sgc":     x509.ExtKeyUsageNetscapeServerGatedCrypto,
}
```

Pour chiffrer les connexions en TLS, on aura surtout besoin de `"signing"`,
`"key encipherment"` et `"server auth"`, pour ajouter l'authenfication par
certificat, il faut aussi `"client auth"`.

### Listes de revocation

Avant de pouvoir utiliser le certificat, il est préférable de générer
la CRL à partir de la base de données. La commande `crl` de `cfssl`
produit la CRL au format PEM sans l'en-tête et le pied de l'armure
ascii, on doit donc arranger la sortie pour obtenir un fichier
exploitable :

```
$ echo "-----BEGIN X509 CRL-----" > crl.pem
$ bin/cfssl crl -db-config ca/db-prod.json -ca ca/ca.pem -ca-key ca/ca-key.pem | fold -w 64 >> crl.pem
$ echo "-----END X509 CRL-----" >> crl.pem
```

On peut examiner le contenu avec openssl :

```
$ openssl crl -in crl.pem -text -noout
```

Par défaut, la CRL est valable 7 jours, l'option `-expiry` permet
définir la durée de validité (en heures avec le suffix `h`
i.e. 7*24=168h).

## Déploiement

Le certificat est prêt à l'emploi, il faut donc :

* Installer le certificat de l'AC et la CRL sur les serveurs et
  clients (`ca/ca.pem` et `crl.pem`)
* Installer le certificat et la clé sur la machine

## Révocation de certificats

Enfin, pour révoquer un certificat, c'est un plus délicat. Il faut
extraire le *serial* du certificat :

```
$ bin/cfssl certinfo -cert pg1.local.pem | jq '.serial_number'
"166108331888504431950509186286490840943777695144",
```

Il faut aussi l'*authority key identifier*, qui doit être en caractères
minuscules et sans les `:`, pour que cfssl trouve le certificat dans la base de
données :

```
$ bin/cfssl certinfo -cert pg1.local.pem | jq '.authority_key_id' | sed -e 's,:,,g;s,.*,\L&,'
"0f572da1471ddc03d0a00e957d49842d382c960a"
```

On peut alors révoquer le certificat dans la base de données :

```
$ bin/cfssl revoke -db-config ca/db-dev.json \
    -serial 166108331888504431950509186286490840943777695144
    -aki 0f572da1471ddc03d0a00e957d49842d382c960a 
    -reason=5
```

Le code pour `-reason` est défini dans la RFC 5280
<https://tools.ietf.org/html/rfc5280#section-5.3.1>, donner une raison
est facultatif.

Dès qu'un certificat est révoqué, il faut regénérer la CRL et la déployer.

On peut spécifier une URL pour la CRL dans ca/config.json, avec la clé
`crl_url` dans chaque profile ou le défaut :

``` json
{
    "signing": {
        "default": {
            "crl_url": "https://.../crl.pem",
            "usages": ...
        }
    }
}
```

L'URL est alors ajoutée dans un champ de *X509v3 CRL Distribution
Points* du certificat.
