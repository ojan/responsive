---
sharing: true
comments: true
layout: post
title:  "Faire suivre les messages Rsyslog vers Logstash"
date:   2014-07-31 08:00:00 +0100
tags:
- logstash
- opensource
- rsyslog
author: Olivier Jan
post_img : ny-times.jpg
lead: "Comment faire suivre les messages Rsyslog vers logstash via JSON."
---

Jusqu'à maintenant, aussi curieux que cela puisse paraître, j'utilisais une méthode plutôt assez « [root](https://github.com/logstash/cookbook/tree/gh-pages/recipes/syslog-pri) » pour faire suivre des logs de type Syslog vers Logstash.

Cette méthode requierait côté Logstash pas mal de « grok parsing » et le résultat n'était pas toujours à la hauteur de mes espérances.

Cett époque est désormais révolue depuis que j'ai lu attentivement le billet en anglais de [Aaron Mildenstein](http://untergeek.com/2012/10/11/using-rsyslog-to-send-pre-formatted-json-to-logstash/) qui propose une approche beaucoup plus simple, puisque Logstash n'a plus qu'à consommer « nativement » les événements.

## Rsyslog, JSON, Logstash

J'ai testé le setup qui suit avec bonheur sur Linux *Ubuntu 14.04.01 LTS 64 bits* avec la version de Rsyslog fournie; soit la *7.4.4* et la dernière version en date de Logstash; soit la *1.4.2-1-2c0f5a*.

Ceci devrait fonctionner sur à peu près toutes les distributions Linux, du moment que celle-ci embarque une version suffisamment récente de Rsyslog capable d'envoyer des logs au format JSON. C'est je crois aux alentours de la version 6 de Rsyslog que cette possibilité est apparue.

### Configuration Rsyslog

Côté Rsyslog, c'est grâce aux templates que nous allons pouvoir pré-formater les messages de façon à ce que Logstash ne fasse plus rien, ou pas grand chose. J'ai placé mon template dans `/etc/rsyslog.d/logstash-json.conf`. En voici le contenu :

~~~
template(name="ls_json"
         type="list"
         option.json="on") {
           constant(value="{")
             constant(value="\"@timestamp\":\"")     property(name="timereported" dateFormat="rfc3339")
             constant(value="\",\"@version\":\"1")
             constant(value="\",\"message\":\"")     property(name="msg")
             constant(value="\",\"host\":\"")        property(name="hostname")
             constant(value="\",\"severity\":\"")    property(name="syslogseverity-text")
             constant(value="\",\"facility\":\"")    property(name="syslogfacility-text")
             constant(value="\",\"programname\":\"") property(name="programname")
             constant(value="\",\"procid\":\"")      property(name="procid")
           constant(value="\"}\n")
         }
~~~

Il est simplement précisé dans ce template de faire coîncider les champs Rsyslog avec ceux choisis côté Logstash. L'`option.json` est bien sûr activée. Merci Aaron pour le boulot fait ! 

Notez au passage le nom du template `ls_json` puisque nous allons maintenant l'appeler depuis le fichier principal de configuration de Rsyslog, situé à `/etc/rsyslog.conf` sur ma distribution. Ajoutez simplement ceci en bas du fichier :

	*.*     @127.0.0.1:10514;ls_json

L'adresse IP est bien sûr à changer si vous n'avez pas comme moi Rsyslog et Logstash sur le même serveur.

Avant de redémarrer Rsyslog par `sudo service rsyslog restart`, passons à la configuration de Logstash.

### Configuration Logstash

L'avantage de ce setup, je me répète, c'est qu'il n'y a plus grand chose à faire, témoigne le peu de longueur et la simplicité de la configuration suivante !

~~~
input {
  udp {
    port => 10514
    codec => "json"
    type => "syslog"
  }
}

filter {
  # This replaces the host field (UDP source) with the host that generated the message (sysloghost)
  if [sysloghost] {
    mutate {
      replace => [ "host", "%{sysloghost}" ]
      remove_field => "sysloghost" # prune the field after successfully replacing "host"
    }
  }
}

output {
  elasticsearch { host => localhost }
}
~~~

En entrée, `input`, une simple écoute UDP sur le port 10514, avec quand même un [codec JSON] histoire de parler le même « protocole ».

Côté filtre, `filter`, c'est juste cosmétique. Il ne serait pas là que cela fonctionnerait quand même. Il permet entre autres de supprimer le champ `sysloghost` une fois correctement remplacé par le nouveau nom de champ `host`, plus générique. Pas de quoi s'affoler !

Et la sortie classique de chez classique vers Elasticsearch. Et oui, Elasticsearch est aussi sur le même serveur !

## Visualisation avec Kibana

Pas non plus à se casser la tête pour visualiser et chercher dans ces logs confortablement depuis son navigateur. Kibana est fait pour ça après tout !

![Tableau de bord Kibana pour Rsyslog Logstash via JSON](../img/posts/rsyslog-json-logstash/kibana-rsyslog-logstash.png)

Et le le clic sur un élément donne tout le détail du message, correctement structuré.

![Détail d'un événement pour Rsyslog Logstash via JSON](../img/posts/rsyslog-json-logstash/kibana-rsyslog-logstash-detail.png)

## Syslog vers Elasticsearch et Kibana

`Ça, c'est fait` sera ma conclusion et cela devient mon setup de référence pour l'ensemble des serveurs que je maintiens en production. Reste plus qu'à mettre ceci en place sur les dits serveurs.