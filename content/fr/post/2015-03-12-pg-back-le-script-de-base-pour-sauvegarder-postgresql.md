---
title: pg_back le script de base pour sauvegarder PostgreSQL
date: 2015-03-12
categories:
- PostgreSQL
tags:
- pg_back
- backup
- dump
---

Il y a fort longtemps, et c'est ma première contribution relative à
PostgreSQL, j'ai écrit un script de backup qui dump tout un serveur
PostgreSQL avec `pg_dump` et `pg_dumpall`. Il s'agit de pg_back.

<!--more-->

Cela peut paraître curieux de publier un simple script de sauvegarde
que tout DBA PostgreSQL a écrit dans sa vie et sait écrire par
cœur. Surtout qu'on le réécrit en permanence ce script, pour ajuster
des chemins, des cas particuliers du serveur à sauvegarder et de
l'environnement où l'on sauvegarde...

En bien justement, c'est parce qu'on le réécrit tout le temps que
pg_back fait gagner du temps. Il est simple et court, facilement
lisible, c'est du shell : tout ce qu'il faut pour en faire une bonne
base pour créer un script de sauvegarde adapté. Quand on l'utilise
comme patron pour en faire un outil plus évolué, on gagne du temps.

Justement rajouter du code pour l'adapter peut se faire au début. Si
on n'a pas envie d'utiliser le fichier de configuration, on adapte la
liste de variables au début du script, quitte à en rajouter.

L'autre endroit intéressant c'est tout à la fin, avant le `exit`, on
peut rajouter tout ce qu'il faut pour externaliser ses sauvegardes.

C'est par [ici](https://github.com/orgrim/pg_back).

