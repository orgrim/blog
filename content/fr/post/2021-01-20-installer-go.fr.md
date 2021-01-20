---
title: "Installer Go"
date: 2021-01-20T21:58:54+01:00
categories:
- Code
tags:
- go
- install
---

Le langage Go est de plus en plus présent, on peut installer son
compilateur et suite d'outil à partir des packages de sa distribution,
mais le plus simple pour avoir une version la plus à jour possible est
d'utiliser les binaires compilés fournis sur le site officiel.

Le version du package golang-go est trop ancienne dans Debian stable
(1.11), je l'ai donc installé de la façon suivante.

<!--more-->

Aller sur <https://golang.org/dl/> pour trouver la dernière version
stable. Pour la suite, on prend l'exemple de la version 1.15.7

```
$ mkdir -p ~/install
$ cd ~/install
$ wget https://golang.org/dl/go1.15.7.linux-amd64.tar.gz
```

Recopier le checksum dans `go1.15.7.linux-amd64.tar.gz.sha256` dans un
format qui convient à `sha256sum` :

```
0d142143794721bb63ce6c8a6180c4062bcf8ef4715e7d6d6609f3a8282629b3  go1.15.7.linux-amd64.tar.gz
```

Vérifier le checksum :

```
$ sha256sum -c go1.15.7.linux-amd64.tar.gz.sha256
go1.15.7.linux-amd64.tar.gz: OK
```

Préparer le répertoire d'installation :

```
$ cd /usr/local/
$ sudo rm -f go
```

Extraire l'archive :

```
$ sudo tar xf ~/install/go1.15.7.linux-amd64.tar.gz
```

Versionner le répertoire et créer un lien symbolique :

```
$ sudo mv go go1.15.7
$ sudo ln -s go1.15.7 go
```

Ajouter à son `~/.bashrc` :

```
PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
export PATH
```

Le sourcer et vérifier la version :

```
$ . ~/.bashrc
$ go version
go version go1.15.7 linux/amd64
```

On peut alors purger les anciennes versions de go dans `/usr/local` ou
revenir en arrière grâce au lien symbolique.

On utilise la configuration par défaut de l'environnement de Go pour
l'installation des binaires et modules avec `go get` dans `~/go`.
