---
title: Rediriger stdout/stderr depuis un script avec du pipe
date: "2011-06-29"
categories:
- Unix
tags:
- unix
- bash
---

Pour rediriger stdout/stderr à l'interieur vers l'entrée standard d'un
commande, il faut utiliser exec et du sous-shell. Cette astuce est un
bashisme a priori.

L'objectif est de renvoyer tous les messages du script dans syslog sans
mettre de redirection sur la ligne de commande. Le principe général
est :

    
    exec FD> >(COMMAND)
    

-   FD est le numéro du file descriptor, 1 pour stdout, 2 pour stderr
-   COMMAND est la commande a exécuter, elle doit bien sûr lire les
    données en entrée.

Un plus gros exemple :

{% highlight bash %}
SYSLOG="no"

# Load configuration file
CONFIG=/etc/myconfig.conf
if [ -f "$CONFIG" ]; then
    . $CONFIG
fi

# Redirect output to syslog if configured
if [ "$SYSLOG" = "yes" ]; then
    SYSLOG_FACILITY=${SYSLOG_FACILITY:-local0}
    SYSLOG_IDENT=${SYSLOG_IDENT:-postgres}

    exec 1> >(logger -t ${SYSLOG_IDENT} -p ${SYSLOG_FACILITY}.info)
    exec 2> >(logger -t ${SYSLOG_IDENT} -p ${SYSLOG_FACILITY}.err)
fi
```

avec dans la configuration :

-   `SYSLOG` : mettre à yes ou no pour activer
-   `SYSLOG_FACILITY` : ou envoyer
-   `SYSLOG_IDENT` : avec quel tag

L'exemple est pris d'un de mes scripts d'archivage pour PostgreSQL, ce
qui permet de logguer dans syslog sans mettre de pipe dans
`archive_command` (ce qu'il ne faut pas faire parce que ça casse : le
code retour donné à PostgreSQL est celui de la dernière commande de la
chaîne de pipe).
