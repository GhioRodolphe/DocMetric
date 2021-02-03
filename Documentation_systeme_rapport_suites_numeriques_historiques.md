## Overview

_**SERVEUR** = Serveur REST implémenté avec un psgi._

### Objectif 
Regrouper tous les métriques collectés en une seule interface et la possibilité de faire des opérations entre chaque métrique.

### Documentation officiel 
* ![Documentation Prometheus](https://prometheus.io/docs/introduction/overview/).
* ![Documentation promQL](https://prometheus.io/docs/prometheus/latest/querying/basics/).
* ![Documentation Net::Prometheus](https://metacpan.org/pod/Net::Prometheus) *(module perl utilisé pour la collecte des données)*.
* ![Documentation  VictoriaMetrics](https://victoriametrics.github.io/).

### Schéma de l'architecture générale
![Schéma de l'architecture générale](/images/general architecture.png)

### Quelques définitions
 * ![Prometheus](https://prometheus.io/docs/introduction/overview/) : 
 Prometheus collecte des données sous forme de séries temporelles. Les séries temporelles sont récupérées de manière active : le serveur Prometheus interroge une liste de sources de données (les exporteurs)
à une fréquence d'interrogation spécifique. Ces points de collecte servent de sources de données à Prometheus. Prometheus dispose d'une API web permettant de visualiser les données collectées.

 * ![PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/) : 
 Prometheus dispose de son propre langage de requête PromQL (Prometheus Query Language) Ce langage permet aux utilisateurs de sélectionner et d'agréger les métriques stockées en base de données.
Il est particulièrement adapté aux fonctionnements avec une base de données de séries temporelles fournissant de nombreuses fonctionnalités spécifiques à la manipulation du temps (décalage de temps, calcul de moyenne, maximum etc.).
Prometheus supporte quatre types de métriques : 
    * ![Jauge ](https://prometheus.io/docs/concepts/metric_types/#gauge) (température absolue, quantité d'espace disque consommé).
    * ![Compteur](https://prometheus.io/docs/concepts/metric_types/#counter) (nombre de requêtes depuis le lancement d'un programme).
    * ![Histogramme](https://prometheus.io/docs/concepts/metric_types/#histogram) (échantillonnage d'un nombre de requête dans plusieurs containers afin de calculer des ![quantiles](https://fr.wikipedia.org/wiki/Quantile)).
    * ![Sommaire](https://prometheus.io/docs/concepts/metric_types/#summary) (relativement similaire à la notion d'histogramme avec des notions supplémentaires).

 * Collecteur : 
 Les collecteurs sont dans notre cas des modules perl. Chacun de ces modules dispose d'une méthode "Create()" et "Collect()". "Create()" retourne un hashmap bénie par le collecteur (.pm). 
"Collect()"  collecte les données et retourne un "![MetricSamples](https://metacpan.org/pod/Net::Prometheus::Types#MetricSamples)". 
Afin de collecter des données il faut que le collecteur soit "créer" ( "Create()" ) et d'être ![enregistrées](https://metacpan.org/pod/Net::Prometheus#register) au sein du **SERVEUR**.
 
### Description générale du fonctionnement côté Citybox
* Les boîtiers sont pourvu d'un **SERVEUR** (/cloudgate/metrics.psgi) qui écoute sur une socket UNIX.
* Le **SERVEUR** est pourvu de plusieurs collecteurs (les collecteurs se trouvent la: /cloudgate/lib/<nom_du_collecteur>.pm).
* Le **SERVEUR** accepte un argument au lancement "SCRAPE_INTERVAL" cette valeur permet de définir l'intervalle de GET de prometheus, en cas d'absence de Prometheus un service Sauron va ![enregistrer](https://metacpan.org/pod/Net::Prometheus#register) un collecteur ( /cloudgate/lib/data_saved_collector.pm  ) qui va récupérer à intervalle de temps régulier les données des collecteurs enregistrés
dans le collecter les données à la place de Prometheus. Une fois que Prometheus est de retour le collecteur va donner les données collectés à Prometheus puis le collecteur va se ![dé-enregistrer](https://metacpan.org/pod/Net::Prometheus#register).
* Le **SERVEUR** met à disposition plusieurs endpoints HTTP pour interagir avec lui, voici une liste de tous les endpoints avec un courte description de ces-derniers:
    *  /prom/metrics : 
        * Endpoint n'acceptant que le User-Agent de Prometheus ("Prometheus/0.00.0") et les méthodes GET.
        * Rôle : Exposer les données à Prometheus.
        * Lorsque le psgi reçoit une requête valide sur cette endpoint le psgi va appeler la fonction ![Collect()](https://metacpan.org/pod/Net::Prometheus#collect) de chacun des collecteurs (collecte des données) pour retourner les données collectées sous le format d'![exposition de Prometheus](https://prometheus.io/docs/instrumenting/exposition_formats/).
        * L'instant pour une requête valide est pris en compte est stocké dans une variable.
    * /complex/insert/<metric_name>/<instance>/<value> :
        * Endpoint n'acceptant que le User-Agent metrics_pusher et les méthodes POST.
        * Rôle : Permettre à l'existant de pousser des données dans Prometheus de manière simple.
    * /simple/insert/<metric_name>/<value> : 
        * Endpoint n'acceptant que le User-Agent "metrics_pusher" et les méthodes POST.
        * Rôle : Permettre à l'existant de pousser des données dans Prometheus de manière simple.
    * /prom/deamon/metrics : 
        * Endpoint n'acceptant que le User-Agent de Prometheus ("Prometheus/0.00.0") et les méthodes GET.
        * Rôle : Exposer les métriques qui ont été poussées par des démons en utilisant le format d'![exposition de Prometheus](https://prometheus.io/docs/instrumenting/exposition_formats/).
    * /prom/status : 
        * Endpoint n'acceptant que le User-Agent "checker" et les méthodes GET.
        * Permet de connaître le statut du psgi.
    * /last/get/prometheus : 
        * Endpoint n'acceptant que le User-Agent "checker" et les méthodes GET.  
        * Rôle : Informer le service sauron l'état des collectes de Prometheus.
        * Permet de connaître le temps écoulé depuis le dernier GET de Prometheus.
       
_Une courte documentation est présente dans les premières lignes du fichier  /cloudgate/metrics.psgi, il y a des exemples de CURL pour tester / debugger._


### Description du fonctionnement coté Citynet
* Chaque groupe possède un serveur Prometheus.
* Étant donnée que la compression de Prometheus n'est pas optimale, une base de données VictoriaMetrics est présente. Prometheus va pousser les données dans Victoria Metrics.
* Rétention Prometheus : ~ quelques jours/semaines.
* Rétention Victoria Metrics: ~ quelques années.
* Des services sauron permettent de gérer les bases de données.
_Une spécification technique est disponible sur ![GitHub](https://github.com/Groupe-Citypassenger-Inc/StageR.GHIO/blob/master/CitynetMetric.md) si tu souhaites avoir plus de détails._

### Description de "Baggage"

Le but de "Baggage" est d'avoir une interface qui centralise l'affichage des données. 