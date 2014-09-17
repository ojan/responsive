---
sharing: true
comments: true
layout: post
title: "Créer un site multilingue avec Jekyll"
date: 2014-07-24 08:00:00 +0200
tags:
- opensource
- jekyll
author: Olivier Jan
post_img: "kust-carte.jpg"
lead: "Comment créer un site multilingue avec Jekyll, le plus à l’aise des générateurs de sites statiques pour cet exercice."
---

Quand nous avons commencé à construire le site de documentation de Check my Website, nous avons cherché un [générateur de sites statiques](/2014/01/generateurs-sites-statiques/) capable de gérer un site multilingue, et à notre grande surprise, nous nous sommes aperçus que ceux-ci ne couraient pas les rues. 

Sans doute dû à une connotation développeur assez forte, les **principaux générateurs se soucient peu de gérer des sites en plusieurs langues**. Après tout, tous les développeurs sont censés lire et écrire l'anglais non !

Nous sommes donc restés avec Jekyll dont la version 2, les [collections](/2014/05/jekyll-collections/) dont nous avons déjà parlé permettent finalement de s'en sortir plutôt proprement. Voici comment implémenter un site multilingue avec Jekyll.

La documentation du service Check my Website allant être GPLv3, vous pouvez consulter le dépôt des sources pour référence sur [Github](https://github.com/checkmyws/help) et même contribuer !

## Le cahier des charges d'un site multilingue

Pour notre site de documentation à venir prochainement, nous avions besoin **de gérer deux langues, l'anglais et le français et de laisser une porte ouverte vers d'autres langues**. Sait-on jamais, le succès de Check my Website pourrait nous amener à traduire cette documentation en d'autres langues. Nous n'en sommes pas là, loin s'en faut !

Le site est découpé en quatres grandes sections :

- Une section FAQ pour; vous l'aurez deviné, une Foire Aux Questions, autrmeent le squestion les plus fréquemment posées.
- Une section Howtos pour écrire des pages qui permettent d'arriver rapidement à tel ou tel résultat.
- Une section Console qui acceuille les pages de documentation de référence de la console.
- Une section API qui accueille les pages de documentation de référence de l'API.

Le plus simple désormais avec Jekyll pour gérer de telles sections est d'utiliser les [collections](http://jekyllrb.com/docs/collections/). **Est-il possible de gérer des collections multilingue** ?

La deuxième difficulté pour un site multilingue est de **ne pas avoir à répéter les `_layouts` Jekyll pour chaque langue**. Il faut donc trouver un moyen pour passer la langue au layout de façon à pouvoir présenter dans chaque langue les chaînes de caractères propres à chacun de ceux-ci.

Enfin, nous voulions créer une page de définitions autour des termes que nous employons dans la documentation. Le plus simple pour ce faire est certainement les `_data` Jekyll. **Mais peut-on les gérer par sous-dossiers, un par langue** ?

## L'implémentation avec Jekyll

Voici le résultat de tout ce cheminement, disons *un résultat* car la nature versatile de Jekyll permet de faire sûrement autrement, peut-être même mieux ?

### Les collections

Les collections doivent bien sûr être déclarées dans le fichier de configuration `_config.yml` situé à la racine d'un projet Jekyll.

~~~
collections:
  faqs:
    output: false
  api:
    output: true
  howtos:
    output: true
  console:
    output: true
~~~

À noter que la collection `faqs` ne génére pas de pages individuelles pour chaque FAQ puisque `output` est positionnée à `false`.

Chaque page de chaque collection utilise le « [front-matter](http://jekyllrb.com/docs/frontmatter/) » Jekyll qui va permettre de placer une variable personalisée `locale` pour chacune des pages à générer.

~~~
---
title: 'howto do this'
locale: 'en'
---
~~~

La variable est bien sûr à positionner à `fr` pour les pages en français, mais cela va de soi !

Pour ne pas voir les pages générées avec un `/en/` ou `/fr/` en fin d'URL, je suis par contre obligé de déclarer explicitement un permalink dans ce même « front-matter » comme ci-dessous :

~~~
---
locale: "en"
permalink: /en/console/website-overview/
---
~~~

C'est un peu dommage mais je n'ai pas réussi à faire autrement, même en jouant sur le paramètre `permalink` au niveau de la collection. Ceci est dû au fait que chaque collection est scindée en sous-dossier par langue.

### Les datas

Pour les fichiers de datas par contre, c'est la classe. À l'instar des collections, nous avons un sous-dossier par langue, soit `_data/en` et `_data/fr`. Dans chaque sous-dossier, un fichier `definitions.yml` contenant les termes et définitions utilisés dans la documentation.

Pour rendre chacune de ces liste de données en fonction de la langue, la boucle suivante est utilisée dans le layout concerné.

{% raw %}
~~~
<dl>
{% assign sorted_defs = site.data.en.definitions | sort: 'word' %}
{% for def in sorted_defs %}
  <dt id="{{ def.word | replace:' ','-' | downcase }}">{{ def.word }}</dt>
  <dd>{{ def.definition }}</dd>
{% endfor %}
</dl>
~~~
{% endraw %}

La boucle est identique pour les deux langues, à l'exception bien sûr de `site.data.en.definitions` pour les définitions en anglais et `site.data.fr.definitions` pour celles en français. Simple et efficace !

### Les pages d'index

Les pages d'index de chaque collection sont directement accessibles à la racine du site, un dossier par langue soit `en/console/index.html` et `fr/console/index.html` par exemple pour l'index des pages de la section console. Ces pages ne contiennent rien d'autres dans mon cas que l'appel au layout pour la section concernée.

### Les layouts, gabarits

C'est ici que la « magie » opère. Pour chaque section, un layout différent. Il est certainement possible de faire un seul layout mais **j'ai préféré partir du principe que chaque section pouvait être présentée de façon différente**.

Par exemple, cela donne pour le layout `console` utilisé par la section du même nom le contenu suivant :

{% raw %}
~~~
---
layout: page
---

{% if page.locale == "fr" %}
  {% assign name = "en français" %}
{% elsif page.locale == "en" %}
  {% assign name = "in english" %}
{% endif %}


{% for article in site.console %}
{% if article.locale == page.locale %}
	<ol>
    	<li class="pages"><a href="{{ article.permalink }}">{{ article.title }}</a><p>{{ article.lead }}</p></li>
    </ol>
   {% endif %}
{% endfor %}
~~~
{% endraw %}

Sont placées **en tête du fichier les assignations de variables locales au layout**, permettant de respecter le principe d'un layout pour toutes les langues.

La suite du fichier est **une boucle itérant sur la collection** console en ne **sélectionnant que les pages ayant une locale égale à la page demandant le layout** : `if article.locale == page.locale`.

### l'arborescence finale du site

En tenant compte du cahier des charges exprimé ci-dessus, voici l'arborescence à laquelle je suis arrivé avec Jekyll :

~~~
_api
_bower_components
_console
_data
_faqs
_howtos
fr
_includes
_layouts
index.md
_plugins
assets
en
~~~

Tous les dossiers commençant par une `_` sont bien sûr des dossiers un peu spéciaux correspondants soit à des dossiers « réservés » Jekyll comme `_data` ou `_layouts`; soit à des collections comme `_api` ou `_faqs`.

Dans chacune des collections, un dossier par langue soit `_console/en` et `_console/fr` par exemple pour les pages de la section console.

## Une documentation multilingue avec Jekyll : C'est possible !

Voilà les bases posées d'un site multilingue avec Jekyll. **L'organisation de l'arborescence permet de bien distinguer chaque langue**, que ce soit au niveau des pages, des collections, des datas et layouts en regroupant chacun de ceux-ci dans un dossier par langue.

Et même si le fait de devoir préciser un `permalink` dans chaque page peut sembler un peu lourd, **le résultat est celui attendu**. Je suis sûr que les concepteurs de Jekyll, qui sont des personnes de grand talent, nous améliorerons ça tôt ou tard !
