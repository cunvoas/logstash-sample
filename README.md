logstash-sample
===============

[[image]]
Que celui qui n'a jamais vécu cette situation peut arrêter la lecture de suite, car je vais tenter ici d'expliquer comment j'ai pu gagner du temps dans ma journée dans l'analyse de mes logs. Les logs, ces fichiers qui semblent structurés mais qui contiennent des informations de tout type et dans lesquel on a souvent du mal à trouver une petite information. 

La solution à mettre à place s'appelle LogStash, elle s'appuie sur ElasticSearch pour stocker et récupérer les données et sur Kibana pour un affichage digne de ce nom.

# Objectif
* Centraliser les données présentes dans les logs
* Analyser les données en temps réél et fournir un tableaux de bord parlant
* Gérer la montée en charge lorsque la journalisation s'accélère

# Installation

Le but ici est d'installer un "Quattuor" permettant la collecte, le traitement et l'affichage des données en assurant une scalabilité horizontale à toute épreuve.
[[image]]

## Logstash (v. 1.4.2)

* Pré-requis : Installation d'une version récente de Java
```bash
> java -version
java version "1.7.0_45"
Java(TM) SE Runtime Environment (build 1.7.0_45-b18)
Java HotSpot(TM) 64-Bit Server VM (build 24.45-b08, mixed mode)
```
* Installation de logstash
Téléchargement de la version [1.4.2](http://www.elasticsearch.org/overview/logstash/download/) : 
```bash
> curl -O https://download.elasticsearch.org/logstash/logstash/logstash-1.4.2.tar.gz
> tar zxvf logstash-1.4.2.tar.gz
> cd logstash-1.4.2
```

Lancement de l'agent logstash (et petit test): 
```bash
> bin/logstash agent -e ''
> A simple message
> {
         "message" => "A simple message\r",
        "@version" => "1",
      "@timestamp" => "2014-11-21T13:55:02.222Z",
            "type" => "stdin",
            "host" => "localhost"
  }
```

Le serveur est donc lancé avec une configuration simple : en entrée stdin() et en sortie stdout(). Donc pour faire simple on envoie des logs dans l'entrée standard de la console, et suite à une analyse de logstash (assez simple pour le coup) la sortie standard reçoit la collecte du résultat.

Une configuration avancée permet de faire un peu plus que la commande echo 'Hello World'. Cette configuration permet de faire transiter un évènement d'un point d'entrée (ajout dans un fichier de log, flux de données des réseaux sociaux, ...) vers un point de sortie (fichiers, base de données, outils de monitoring) en ayant été aggrgé par un ensemble de filtres entre temps.

La configuration est découpée en 4 axes principaux :
* Inputs : Les entrées pour collecter les données
* Outputs : Les tuyaux de sortie
* Filters : ...
* Codecs : ...

## ElasticSearch (v. 1.1.1)

* Pré-requis : Aucun

* Installation de elasticsearch
Chaque release de logstash sort avec une recommandation concernant les versions d'outillage à mettre en relation avec lui. Ici il est donc conseillé d'utiliser la version 1.1.1 d'elasticsearch.
Nous allons ici installer une version standard d'elasticsearch sans rentrer dans les détails dont de plus amples informations sont disponibles sur le [site](http://www.elasticsearch.org/guide/en/elasticsearch/reference/index.html).
```bash
curl -O https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.1.1.tar.gz
tar zxvf elasticsearch-1.1.1.tar.gz
cd elasticsearch-1.1.1/
```

* Lancement du serveur
```bash
./bin/elasticsearch
```
Le serveur tourne alors sur le port 9200 et il est alors facile de l'interroger soit par exemple via [curl](http://curl.haxx.se) en ligne de commande, soit via des plugins comme [elasticsearch-head](https://github.com/mobz/elasticsearch-head).
Il peut désormais recevoir toutes les données à collecter puis servir de backend pour logstash.

## RabbitMQ (v. 3.2.3)
Le serveur de messaging va permettre de procéder à un mode asynchrone pour la collecte des données, et ainsi réduire le nombre d'opération bloquante pour logstash.
ElasticSearch ira consommer les messages à son bon vouloir, et poourra même se permettre d'être indisponible sans qu'un message ne soit perdu (il sera stocké dans RabbitMQ).

* Pré-requis :
ERLANG

* Installation

* Lancement du serveur

# Configuration

### Configuration minimale
Nous allons tout d'abord commencer par définir un fichier de configuration pour logstash puis le passer en paramètre de l'agent.
Essayons alors de mettre en input l'entrée standard et en output la sortie standard.

```bash
# Différentes sources d'entrée
input {
	# Entrée standard
	stdin { }
}
# Filtres appliqués
filters {
	
}
# Différents flux de sortie
output {	
	# Sortie standard
	stdout { debug => true }
}
```
Lançons désormais l'agent en passant le fichier de configuration en paramètre
```bash
> bin/logstash agent -f /var/data/logstash/conf/logstash.conf
> A second message
> {
         "message" => "A second message\r",
        "@version" => "1",
      "@timestamp" => "2014-11-21T15:56:01.228Z",
            "type" => "stdin",
            "host" => "localhost"
  }
```

### Collecte des données d'un fichier de log
Il faut désormais indiquer à logstash de collecter les données non pas de l'entrée standard mais à partir des logs d'un serveur (ex: log d'accès apache)
```bash
# Différentes sources d'entrée
input {
	# Log Apache ACCESS
	file {
		path => "/var/log/server/apache/access.log"		
		type => "apache.access"
	}
}
...
```
En indiquant un type à ces données collectées, il sera facile par la suite de travailler avec.

### Filtrage des données collectées
Les données sont brutes, il faut donc indiquer quel pattern elle respecte pour pouvoir ranger la bonne information dans la bonne case. Logstash fournit un ensemble de pattern de base, et utilise la librairie [grok](https://code.google.com/p/semicomplete/wiki/Grok) pour le parsing. 
Une bonne application pour parser vos data : [http://grokdebug.herokuapp.com].

Prenons l'exemple de ce fichier de [log](http://monfichierdelog)
On remarque bien la séparation entre chaque type d'information.
Cela permet d'ajouter un filtre dans la configuration:
```bash
filter {
	if [type] == "apache.access" {
		grok {
			# Se basant sur le fichier de pattern grok-pattern
			match => { "message" => "%{APACHE_ACCESS_LOG}" }
			add_tag => ["access"]						
		}	
	}
}
```

Et de compléter ma liste de pattern pour grok
```bash
HTTPVERSION (HTTP\/%{NUMBER})
HTTPMETHOD (GET|POST)
APACHEACCESSDATESTAMP %{MONTHDAY}/%{MONTH}/20%{YEAR}:%{HOUR}:%{MINUTE}:%{SECOND}
APACHE_ACCESS_LOG %{IPORHOST:clientip}%{SPACE}\[%{APACHEACCESSDATESTAMP:timestamp}\]%{SPACE}\"%{HTTPMETHOD:httpmethod} %{URIPATHPARAM:url} %{HTTPVERSION:httpversion}\"%{SPACE}%{NUMBER:httpcode}%{SPACE}%{NUMBER:httpresponse}
...
```

En redémmarrant l'agent les modifications seront prises en comptes et chaque ligne ajoutée dans le fichier sera alors filtrée puis envoyée dans la sortie standard.
```bash
> bin/logstash agent -f /var/data/logstash/conf/logstash.conf
> {
    "message": "64.242.88.10\t[08/Mar/2004:01:47:06]\t \"GET /store/ipad/info HTTP/1.1\" 401 1284\r",
    "@version": "1",
    "@timestamp": "2014-11-20T15:32:24.186Z",
    "type": "log4j",
    "host": "localhost",
    "clientip": "64.242.88.10",
    "timestamp": "08/Mar/2004:01:47:06",
    "httpmethod": "GET",
    "url": "/store/ipad/info",
    "httpversion": "HTTP/1.1",
    "httpcode": "401",
    "httpresponse": "1284",
    "tags": [
      "access"
    ]      
  }
```
 
### Ajout de catégories

Logstash permet de tagguer certaines requêtes, ce qui permet de les retrouver plus facilement par la suite.
Pour cela, il suffit d'ajouter un tag selon dans telle ou telle condition.
```bash
	# Info vs PreCommand
	if [url] =~ "\/info$" {
		mutate {
			add_tag => ["info"]
		}
	} else if [url] =~ "\/precommand$" {
		mutate {
			add_tag => ["precommand"]
		}
	}
```

### Intégration des données de géolocalisation

Le filtre de géolocation [geoip]() permet d'intégrer facilement à partir de l'adresse IP fournie dans un champs les données de positionnement d'où provient l'hôte ayant effectué la requête. Grâce à cela, il est possible d'intégrer dans le tableau de bord la location précise d'un utilisateur.

Pour cela il faut ajouter les filtres geoip et mutate
```bash
filter {
	if [type] == "apache.access" {
		grok {
			# Se basant sur le fichier de pattern grok-pattern
			match => { "message" => "%{APACHE_ACCESS_LOG}" }
			add_tag => ["access"]						
		}
		mutate {
			convert => [ "httpreponse", "integer" ]
			convert => [ "[geoip][coordinates]", "float" ]
		}
		geoip {
			source => "clientip"
			target => "geoip"
			add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
			add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
		}
	}
}
```

Relançons l'agent logstash avec cette nouvelle configuration, et voyons que les données en output sont complétées avec des informations de géolocalisations.
```bash
> bin/logstash agent -f /var/data/logstash/conf/logstash.conf
> {
    "message": "64.242.88.10\t[08/Mar/2004:01:47:06]\t \"GET /store/ipad/info HTTP/1.1\" 401 1284\r",
    "@version": "1",
    "@timestamp": "2014-11-20T15:32:24.186Z",
    "type": "log4j",
    "host": "localhost",
    "clientip": "64.242.88.10",
    "timestamp": "08/Mar/2004:01:47:06",
    "httpmethod": "GET",
    "url": "/store/ipad/info",
    "httpversion": "HTTP/1.1",
    "httpcode": "401",
    "httpresponse": "1284",
    "tags": [
      "access"
    ],
    "geoip": {
      "ip": "64.242.88.10",
      "country_code2": "US",
      "country_code3": "USA",
      "country_name": "United States",
      "continent_code": "NA",
      "region_name": "CA",
      "city_name": "San Francisco",
      "postal_code": "94107",
      "latitude": 37.7697,
      "longitude": -122.39330000000001,
      "dma_code": 807,
      "area_code": 415,
      "timezone": "America/Los_Angeles",
      "real_region_name": "California",
      "location": [
        -122.39330000000001,
        37.7697
      ],
      "coordinates": [
        "-122.39330000000001",
        "37.7697"
      ]
    }
  }
 ```

Remarques : Logstash utilise la base de données GeoLite de Maxmind pour localiser une adresse IP. Cela est configurable dans le filtre pour utiliser une autre base de données.

### Affichage via le DashBoard dans Kibana

Pour lancer Kibana, le module web utilisé pour afficher les données dans un tableau de bord, il suffit de lancer la commande 
```bash
> bin/logstash web
```
Le tableau de bord est disponbile à (http://localhost:9292/index.html#/dashboard/file/logstash.json). Celui est est modifiable à souhait, il est également possible de l'exporter ou d'en importer.
Celui-ci expose 2 types d'information : la fréquence d'arrivée d'évènement et les données traitées.

[[image]]
[[image]]

Il suffit désormais d'afficher les informations de géolocalisation via la configuration du DashBoard (en haut à droite de la page).
Ajoutons dans un premier temps une ligne (onglet Rows) qui contiendra notre Carte de géolocalisation. Puis en revenant sur la DashBoard il est possible d'aller configurer cette nouvelle ligne en y ajoutant un cadre (Panel) de type "bettermap". Ce cadre a uniquement besoin des coordonnées pour situer la provenance des appels.

[[image]]

Nous pouvons également ajouter la répartition des appels effectués se différençiant par les tags recemment créés.

[[image]]

# Haute disponibilité
Avec cette architecture, nous voyons bien le rôle important que joue ElasticSearch en tant que composant backend de logstash.
En cas d'interruption de service de sa part les logs de sont plus disponibles pendant la coupure de service, mais le plus grave est que les logs ne sont plus collectés. Il est donc indispensable de prévoir un mode asynchrone pour la collecte.
++ répartition de la charge


# Démonstration


