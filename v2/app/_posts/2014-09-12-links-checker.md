---
sharing: true
comments: true
layout: post
title:  "Vérifier les liens morts sur un site web"
date:   2014-09-12 08:00:00 +0100
tags:
- opensource
- monitoring
author: Olivier Jan
post_img : reward-deadlinks.png
lead: "Difficile de trouver un vérificateur de liens morts sur un site. Récit de l'aventure vécue !"
---

Quelle n'a pas été ma surprise en essayant de trouver un vérificateur de liens pour le tout [nouveau site de support](http://help.checkmy.ws) de voir à quel point ce simple besoin pouvait se transformer en quête du Graal.

Entre les projets Open Source abandonnés et les nombreux sites en ligne qui ne vérifient qu'une page, c'est à croire que plus **personne ne vérifie les liens internes et externes de son site** pour contrôler s'ils sont toujours valides ou morts.

Et bien moi si ! je trouve que c'est une partie importante du confort de lecture offert au visiteurs. **Qui aime cliquer un lien alléchant pour s'apercevoir qu'il ne mène nul part** ?

## Cahier des charges

Pourtant mon cahier des charges pour un outil de ce genre est plutôt basique.

- Capacité à tester un site entier en une seule commande (récursif).
- Automatisable.
- HTTP et HTTPS.
- Ignorer les erreurs de certificat SSL.
- Support HTML5.
- Respect du fichier robots.txt.

## Les vérificateurs de liens morts

Voyons le résultat de mes recherches tout azimut pour trouver ce qu'il va convenir d'appeler la perle rare. Vous allez comprendre pourquoi.

### Librairies

Vu que nous utilisons un workflow de publication basé entre autres sur [Grunt et Bower](/2013/12/yeoman-grunt-bower-workflow/), il semblait être une bonne idée de chercher quelque chose du côté des librairies, [plugins](http://gruntjs.com/plugins) pour Grunt. Deux possibilités existent mais ça ne va pas bien loin.

Jamais réussi à faire fonctionner correctement [grunt-deadlink](https://www.npmjs.org/package/grunt-deadlink) et [grunt-link-checker](https://www.npmjs.org/package/grunt-link-checker) sort constamment avec une erreur. Ça commence fort !

Pas plus de chance avec un module npm du nom de [hyperlink](https://www.npmjs.org/package/hyperlink). La loose gagne du terrain.

### Services en ligne

Le plus célèbre est certainement celui du [W3C](http://validator.w3.org/checklink). Il est malheureusement **gratuit uniquement pour une page web et non un site entier** donc pas ce que je recherche.

[Broken Link Check](http://www.brokenlinkcheck.com/) est le service en ligne gratuit qui m'a semblé correspondre le mieux à ce que je recherchais. L'outil est limité aux 3000 pages rencontrées, ce qui en fait l'un des moins limités que j'ai trouvé dans ce domaine des outils en ligne.

Et la sortie, succinte, présente l'essentiel; à savoir les pages contenant des liens morts et la possibilité de facilement les retrouver comme témoigne cet écran.

![Interface de Broken Link Check](../img/posts/links-checker/brokenlinkcheck-interface.png)

Mais bon côté automatisation, sans API, c'est pas le pied un service en ligne !

### Logiciels dédiés

Reste un logiciel dédié pour me sortir de ce qui commence à être une vraie prise de tête !
La fraîcheur de ce qu'il est possible de trouver en ce domaine fait penser que ma quête semble avoir été résolue depuis les années 2000 et que rien n'a bougé depuis. 

Deux logiciels sortent du lot et semblent mieux suivis que leurs petits camarades.

#### Webcheck

La commande à exécuter peut rester simple, même si pas mal d'options sont disponibles :

	webcheck.py http://wooster.checkmy.ws

Dernière sortie en 2010 pour ce [logiciel](http://arthurdejong.org/webcheck/) en Python qui fonctionne plutôt bien et a certainement la sortie la plus complète comme en témoignent les deux captures ci-dessous.

![Vue d'ensemble du site vérifié avec Webcheck](../img/posts/links-checker/webcheck-overview.png)

![Vue des liens morts ou posant problème dans Webcheck](../img/posts/links-checker/webcheck-deadlinks.png)

C'est pas vraiment sexy mais plutôt efficace pour pister et retrouver ces foutus liens morts.

#### LinkChecker

Ce logiciel se présente à la fois comme une ligne de commande, une GUI et une interface web, plutôt un bon début. Et en plus, le projet reste actif malgré le nombre d'issues remontées restant sans réponse. La dernière version est datée de cette année.

La grande force de LinkChecker est la multitude des formats de sorties disponibles, qui vont du plus classique `html`, `csv` au plus exotique comme `dot`, qui génère un graphique au format dot, celui de [GraphViz](http://fr.wikipedia.org/wiki/Graphviz).

Voici un exemple de la commande avec une sortie HTML :

	linkchecker --check-extern -F html http://wooster.checkmy.ws/

Dans la page `linkchecker-out.html` générée par l'exécution du script, nous trouvons des blocs similaires à celui-ci qui permettent de voir à la fois l'URL en erreur et la page contenant cette erreur. Pratique pour retrouver facilement dans le code source l'URL incriminée.

<table class="table table-bordered"><tr>
<td class="url">URL</td>
<td class="url">`https://docpad.org/docs/'</td></tr>
<tr><td>Name</td><td>`documentation'</td></tr>
<tr><td>Parent&nbsp;URL</td><td><a target="top" href="http://wooster.checkmy.ws/2014/01/docpad/">http://wooster.checkmy.ws/2014/01/docpad/</a>, line 119, col 925
(<a href="http://validator.w3.org/check?ss=1&amp;uri=http://wooster.checkmy.ws/2014/01/docpad/">HTML</a>)
(<a href="http://jigsaw.w3.org/css-validator/validator?uri=http://wooster.checkmy.ws/2014/01/docpad/&amp;warning=1&amp;profile=css2&amp;usermedium=all">CSS</a>)</td></tr>
<tr><td>Real&nbsp;URL</td><td><a target="top" href="https://docpad.org/docs/">https://docpad.org/docs/</a></td></tr>
<tr><td>Check&nbsp;time</td><td>374.962 seconds</td></tr>
<tr><td class="error">Result</td><td class="error">Error: ConnectionError: HTTPSConnectionPool(host='docpad.org', port=443): Max retries exceeded with url: /docs/ (Caused by &lt;class 'socket.error'&gt;: [Errno 101] Network is unreachable)</td></tr>
</table>

Cette URL est bien en erreur puisque le site ne répond plus en HTTPS, mais seulement en HTTP. Par contre, au chapitre grief, les liens sortent en erreur quand le certificat SSL n'est pas valide.

### Old school Bash script

J'ai d'abord essayé avec une commande `wget` un peu tarabiscotée :

	wget --spider -o wget.log -e robots=off --wait 1 -r -p http://www.example.com

Mais ça ne donne rien de vraiment fiable. On oublie.

Puisque je ne trouve rien à ma convenance, cette histoire va se finir par un script bash s'appuyant sur `Lynx` et `Curl`. J'ai trouvé ce script dans le [Linux Shell Scripting Book](https://www.packtpub.com/hardware-and-creative/linux-shell-scripting-cookbook-second-edition) et l'ai un peu modifié pour ignorer les erreurs de certificat.

~~~
#!/bin/bash
#Filename: find_broken.sh
#Desc: Find broken links in a website

if [ $# -ne 1 ];
	then
	echo -e "usage: $0 URL\n"
	exit 1;
fi

echo Broken links:

mkdir /tmp/$$.lynx
cd /tmp/$$.lynx

lynx -traversal $1 > /dev/null
count=0;

sort -u reject.dat > links.txt

while read link;
do
	output=`curl -I -k $link -s | grep "HTTP/.*200"`;
	if [[ -z $output ]];
		then
		echo $link;
		let count++
	fi
done < links.txt

[ $count -eq 0 ] && echo No broken links found.
~~~

C'est « cochon » mais ça fait plutôt bien le job si l'on excuse la relative lenteur des vérifications. Le script considère les redirections comme des erreurs.

## Alors lequel ?

Il existe certainement d'autre logiciels, services pour contrôler les liens morts d'un site web mais ils sont sûrement bien cachés et du coup je ne les ai pas trouvés.

Dans ceux présentés, mon choix, non définitif, ira pour le moment à ce bon vieux script bash, suivi du service en ligne Broken Link Checker et enfin de LinkChecker, en attendant que celui-ci puissent ignorer les erreurs de certificat. Quelque chose me dit que **la détection de liens morts est une fonction que nous pourrions trouver un jour dans [Check my Website](http://www.checkmy.ws)… Mais en plus sexy quand même j'espère** !
