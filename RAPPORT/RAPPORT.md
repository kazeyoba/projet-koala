# Titre

# Plan d'adressage

Dans un premier temps, nous avons réalisé un plan d’adressage de manière à créer deux sous-réseau pour le VLAN front et back.

| Nom            | VLAN Front    | VLAN Back     |
| -------------- | ------------  | --------------|
| IP sous réseau | 10.10.60.0    | 10.10.61.0    |
| Masque         | 255.255.255.0 | 255.255.255.0 |
| Passerelle     | 10.10.60.254  | 10.10.61.254  |
| Broadcast      | 10.10.60.255  | 10.10.61.255  |


## Architecture Réseau

Maintenant que nous avons le plan d’adressage, il nous est désormais possible de réaliser la modélisation de l’architecture.

![Texte alternatif](schema-architecture-koala.png)

Comme illustré dans le schéma ci-dessus, notre réseau est divisé en deux sous-réseaux distincts : le front et le back.

Dans le front, on retrouve :

* 2 proxy / load-balancers, ils ont pour roles de répartir la charge entre les différents serveurs, assurant ainsi une distribution équilibrée des requêtes. Cette fonction est mise en œuvre grâce à l'utilisation de HAProxy et de keepalived.
* 2 serveurs web, avec du apache
* 1 serveur grafana pour surveiller et à la visualiser les performances du réseau et des applications

Puis dans le back, on retrouve :

* 1 serveur ansible sur lequel on retrouve nos playbooks
* 2 serveurs de bases de données, mariadb
* 1 serveur loki pour l''agrégation et à l'analyse des journaux
* 1 serveur NFS sur lequel on retrouve la configuration de kanboard

## Démonstration

### Execution playbook ansible

[Execution playbook ansible](https://www.youtube.com/watch?v=kEErIgz4k6g)

### MariaDB Galera 

[Creation BDD + Replication](https://youtu.be/mSk_c7yrza4)

### Kanboard & HA Proxy

[HAProxy & Kanboard](https://www.youtube.com/watch?v=QUFcWhP8TWw)

### Playbook opnsense

[Execution playbook](https://youtu.be/SZFz1TTxgcA)

### Serveur NFS

## Tableau de bord Grafana 

![grafana5](https://github.com/kazeyoba/projet-koala/assets/64216801/0fb10889-0c25-45be-ad1d-a523fe897c54)

### Expression loki(LogQL) utiliser

* {job="apache"}
* Le nombre de requêtes effectuées par IP -> Apache-00 -> sum by (client_ip) (count_over_time({job="apache", host="web-00.koala-60.cloud"}[1m])) -> Apache-01 -> sum by (client_ip) (count_over_time({job="apache", host="web-01.koala-60.cloud"}[1m]))
* Une statistique sur les codes de retour HTTP  -> count_over_time({job="apache"} |= '404' [$__auto]) -> le camenbert qui remonte le % de code reponse http 404 sur les 2 apaches | Vert web-00 | Jaune web-01.
* Une statistique sur les codes de retour HTTP  -> Apache-00 -> sum by (response_code) (count_over_time({job="apache", host="web-00.koala-60.cloud"}[1m])) -> Apache-01 -> sum by (response_code) (count_over_time({job="apache", host="web-01.koala-60.cloud"}[1m]))
* Une statistique sur les URLs les plus visitées -> Apache-00 -> topk(10, sum by(request_path) (count_over_time({job="apache", host="web-00.koala-60.cloud"}[24h]))) Apache 01 -> topk(10, sum by(request_path) (count_over_time({job="apache", host="web-01.koala-60.cloud"}[24h])))

### Configuration YAML d'un des Promtail sur le serveur Apache-00 :

server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

client:
  url: http://10.10.61.30:3100/loki/api/v1/push

scrape_configs:
  - job_name: journal
    journal:
      max_age: 1h
      path: /var/log/journal
      labels:
        job: systemd
        host: "web-00.koala-60.cloud"
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'
  - job_name: apache
    static_configs:
      - targets:
          - localhost
        labels:
          job: apache
          __path__: /var/log/apache2/*.log
          host: "web-00.koala-60.cloud"
    pipeline_stages:
      - match:
          selector: '{job="apache"}'
          stages:
            - regex:
                 expression: '^(?P<client_ip>[^ ]*) - - \[(?P<timestamp>[^\]]*)\] "(?P<request_method>[A-Z]+) (?P<request_path>[^"]+) HTTP/(?P<http_version>[0-9.]+)" (?P<response_code>\d+) (?P<response_size>\d+) "(?P<referrer>[^"]*)" "(?P<user_agent>[^"]*)"$'
            - labels:
                client_ip:
                request_method:
                request_path:
                http_version:
                response_code:
                response_size:
                referrer:
                user_agent:

        ## Explications :

        #serveur :

        http_listen_port : C'est le port sur lequel Promtail va écouter pour le trafic HTTP.
        grpc_listen_port : C'est pour le gRPC, mais ici c'est mis à zéro donc on ne l'utilise pas.

        #Configuration du client :
        Url : C'est l'adresse de notre instance Loki où Promtail va envoyer les logs.

        #Configurations de récupération (scraping) :
        job_name : On donne un nom à notre job pour pouvoir le repérer plus facilement.
        static_configs : Ici, on définit des cibles statiques pour notre job, avec des labels spécifiques.
        journal : Ça c'est pour récupérer les logs du journal de systemd.
        path : Le chemin vers les fichiers de log qu'on veut récupérer.
        relabel_configs : Ça nous permet de modifier les labels des logs dynamiquement.

        #Étapes de traitement (pipeline stages) :
        match : Cette étape filtre les logs en utilisant le sélecteur fourni.
        regex : On utilise des expressions régulières pour extraire des champs spécifiques des logs.
        labels : Les champs extraits sont transformés en labels pour le flux de logs.
        Dans notre cas, on extrait pas mal d'infos des logs Apache, comme l'IP du client, la méthode de requête, le chemin, la version HTTP, le code de réponse, la taille de la réponse, le référent et l'agent utilisateur.

        Cette configuration nous permet de faire des requêtes sur nos logs dans Loki en utilisant ces labels, ce qui facilite grandement le filtrage et la recherche dans nos données de logs. Par exemple, on peut chercher tous les logs qui ont un code de réponse spécifique, ou voir toutes les requêtes faites par une certaine IP client.

        


