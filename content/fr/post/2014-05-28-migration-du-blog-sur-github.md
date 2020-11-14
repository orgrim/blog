---
title: Migration du blog sur github
date: "2014-05-28"
categories:
- Unix
tags:
- blog
- pandoc
---

Finalement le temps ne me permettant plus d'administrer mon serveur
dédié comme il faut, j'ai décidé de l'abondonner et transférer mes
services ailleurs. Pour le blog et le wiki, j'ai choisi de faire un
site/blog github avec jekyll, vu que c'est tout prémaché.

<!--more-->

Sur github, l'hébergement c'est du fichier html, donc statique, on a
plusieurs possibilités pour créer un site :

* [jekyll] dont github est capable de compiler le résultat à chaque
  push du dépôt, mais avec certaines limitations, comme aucun plugin
  et la nécessité de créer pas mal de pages pour que ça ressemble à un
  blog
* [octopress] qui est une surcouche à jekyll, il faut rtfm et tout de
  même faire ses templates, voir un peu de ruby
* [pelican] qui fait comme octopress à peu prêt mais en python
* plein d'autres générateurs de site statiques que j'ai surement raté


L'avantage de pelican est dans sa capacité à importer le contenu d'un
blog dotclear, et transformer la syntaxe en markdown. Le markdown
semble avoir la cote, beaucoup de générateurs de site l'utilisent,
d'ailleurs je m'y suis mis.

Comme j'avais déjà fait le site de [pitrery] avec jekyll sur github,
la migration fut plus facile, une fois le dotclear migré grâce à
pelican.

Au final, j'ai maintenant un truc tout simple avec un thème
[bootstrap], et le jekyll de base de github est largement suffisant.

La bonne découverte fut [pandoc], un super convertisseur de format de
document texte, qui va me servir pour la migration du dokuwiki à base
de :

    pandoc -f html -t markdown http://wiki.orgrim.net/page?do=export_html

Le mot de la fin, ça demande un peu de boulot de nettoyage manuel,
mais vu le nombre impressionant de posts que j'ai pu produire en quatre
ans, c'était pas trop méchant.

[jekyll]: http://jekyllrb.com
[octopress]: http://octopress.org
[pelican]: http://getpelican.com
[pitrery]: http://dalibo.github.io/pitrery/
[bootstrap]: http://getbootstrap.com
[pandoc]: http://johnmacfarlane.net/pandoc/
