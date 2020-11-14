---
title: Combiner des PDF en un seul
date: "2011-06-21"
categories:
- Unix
tags:
- unix
- ghostscript
- pdf
---

Pour combiner des pdf en un seul, on peut essayer `pdfjoin` fournit par
le projet [pdfjam]. En attendant que les 250 Mo de dépendances (Latex
principalement) s'installent, on peut utiliser ghostscript :

    
    gs -sDEVICE=pdfwrite -dNOPAUSE -dQUIET -dBATCH -sOutputFile=../combined_doc.pdf *.pdf
    

Merci à [perlmonks.org].

[pdfjam]: http://www2.warwick.ac.uk/fac/sci/statistics/staff/academic-research/firth/software/pdfjam/ "pdfjam"
[perlmonks.org]: http://www.perlmonks.org/?node_id=470323
