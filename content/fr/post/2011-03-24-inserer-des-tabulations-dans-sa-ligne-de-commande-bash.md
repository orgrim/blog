---
title: Insérer des tabulations dans sa ligne de commande bash
date: "2011-03-24"
categories:
- Unix
tags:
- unix
- bash
---

Il y a longtemps que je me demandais comment faire ça, sans prendre la
peine de rechercher ou lire le man. C'est chose faite, pour insérer une
tabulation, il faut contourner la complétion de commandes avec le combo
suivant :

    
    C-v TAB
    

C'est simple, mais ça ne s'invente pas.

Pour le coup, c'est utile quand on veut voir un fichier de configuration
sans les commentaires, par exemple` postgresql.conf` :

    
    grep -Ev '^(( |        )*#|$)' postgresql.conf
    

On doit donc taper une tabulation dans l'expression régulière : on
affiche toutes les lignes qui ne satisfont pas la condition « tout ce
qui commence par 0 ou plusieurs espaces ou tabulations suivi d'un dièse,
ou une ligne vide ».

