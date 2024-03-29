# Projet infrastructure ansible

## Plan d'adressage

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

## Playbooks

### All

Pour l'ensemble des machines de notre réseau, nous avons configuré ssh de manière à pouvoir si connecté à distance : 

```yml
---
- name: Add my ssh key
  authorized_key:
    user: cloud
    state: present
    key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCl9XR5SgQXbK6kzbe53HVvxCw4c26hWsxC28kzGpMewSCPE64TUVa2UyXcgdiNQOHYuyCBSllk0sblp+iPKpf4t2vvzilS0nowoCU6XWqanLiakGjUgro0xjwdAMWNaIARKnOl4QoIuWLxs0duGhK+z63AQMi33QuG8VbCV2vD5KRhunsM2bN3MK1H6IpeBeteScMxnvk5EasG7VTesCIzsbYqj5RWJ+7MJEHonOUm798/5lHblHPG7+dVTJm06OrjleL0TQZSxhkIy7V7Ho28JHqDaZsZKwyVNUtdaVhOxBHMUjwnDQ4gJAA98pIoFZDuBe31SoXkeb21Hi9Q8GnB cloud@bastion-ans"

- name: Configure SSHd disable PasswordAuthentication
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "^#PasswordAuthentication yes"
    line: "PasswordAuthentication no"

- name: Vérifier si la ligne existe déjà dans le fichier sudoers
  shell: "grep -q 'cloud ALL=(ALL) NOPASSWD:ALL' /etc/sudoers"
  ignore_errors: yes
  register: grep_result

- name: Ajouter la ligne au fichier sudoers si elle n'existe pas
  lineinfile:
    path: /etc/sudoers
    line: 'cloud ALL=(ALL) NOPASSWD:ALL'
  when: grep_result.rc != 0

- name: restart SSHD
  command: systemctl restart sshd
```

### Keepalived

Pour notre master:
```yml
---
- name: Install Keepalived
  apt:
    name: keepalived
    state: latest
- name: Copy keepalived master configuration
  template:
    src: roles/keepalived/templates/master.cfg
    dest: /etc/keepalived/keepalived.conf
- name: restart keepalived
  systemd:
    name: keepalived.service
    state: restarted
```

Pour notre backup:
```yml
---
- name: Install Keepalived
  apt:
    name: keepalived
    state: latest
- name: Copy keepalived backup configuration
  template:
    src: roles/keepalived/templates/backup.cfg
    dest: /etc/keepalived/keepalived.conf
- name: restart keepalived
  systemd:
    name: keepalived.service
    state: restarted
```

Il y a une différence entre la configuration du backup et master par le STATE et priority qui définit la priorité de celui qui prend l'IP Failover. Si le master tombe c'est le serveur secondaire qui prendra l'IP.

### Proxy

```yml
---
- name: make non local bind
  sysctl:
    name: net.ipv4.ip_nonlocal_bind
    value: "1"
    state: present
    reload: yes

- name: install haproxy
  apt:
    name: haproxy
    update_cache: yes
    state: latest

- name: install hatop
  apt:
    name: hatop
    update_cache: yes
    state: latest

- name: Copy haproxy configuration file
  template:
    src: roles/proxy/templates/haproxy.cfg
    dest: /etc/haproxy/haproxy.cfg

- name: Reload haproxy
  systemd:
    name: haproxy.service
    state: reloaded
```
Pour rendre nos BDD & serveur WEB en haute disponibilité on utilise HAProxy pour loadbalancer. On expose nos services en utilisant l'IP Failover.

Voici la configuration:

```conf
frontend B3-front
	bind 10.10.60.103:80
	default_backend B3-backend

backend B3-backend
	balance leastconn
	option httpclose
	option forwardfor
	compression algo gzip
	server web-00.koala-60.cloud     10.10.60.20:80 check
	server web-01.koala-60.cloud     10.10.60.21:80 check

listen SQLbackend
    balance leastconn
    bind 10.10.60.103:3306
    mode tcp
    option tcplog
    server bdd-00.koala-61.cloud 10.10.61.10:3306 check
    server bdd-01.koala-61.cloud 10.10.61.11:3306 check
```

On indique qu'il faut écouter sur l'ip failover.

### Base de données

Pour la base de données on utilise un cluster MariaDB avec Galera pour avoir un cluster "MASTER MASTER". Les bases de données sont synchronisées à la volée.

```yml
---
- name: Install MariaDB & dependances
  apt:
    pkg:
      - mariadb-server
      - mariadb-client
      - python3-pip
      - pkg-config
      - libmariadb-dev
      - rsync
    state: latest
    update_cache: yes

- name: Install mysclient client for Python 3.X
  shell: pip3 install mysqlclient --break-system-packages

- name: modify mariadb configuration file
  lineinfile:
    path: /etc/mysql/mariadb.conf.d/50-server.cnf
    regexp: "^bind-address.*$"
    line: 'bind-address = 0.0.0.0'
  register: mariaconf

- name: restart mariadb
  systemd:
    name: mariadb.service
    state: restarted
  when: mariaconf.changed

- name: Créer une base de données et un utilisateur
  mysql_db:
    name: "kanboard"
    state: present

- name: Créer un utilisateur et accorder des privilèges
  mysql_user:
    name: "kanboard"
    host: "%"
    password: "kanboard"
    priv: "kanboard.*:ALL"
    state: present
```

> Point de subtilité pour faire les opérations sur une base de données et avec les dernières versions de Python3 et pip on doit le faire par le shell car on doit ajouter l'argument --break-systems-packages.

#### Galera

Pour le premier "MASTER":
```yml
---
- name: Verification si une configuration existe
  stat:
    path: '/etc/mysql/conf.d/galera.cnf'
  register: check_galeracnf

- name: Copy Galera bootstrap configuration file
  template:
    src: roles/galera/templates/galera_master.cnf
    dest: /etc/mysql/conf.d/galera.cnf
  register: galeracnf
  when: check_galeracnf.stat.exists == False
  
- name: Restart mariadb
  systemd:
    name: mariadb.service
    state: restarted
  when: galeracnf.changed
```

Pour le backup:
```yml
---
- name: Verification si une configuration existe
  stat:
    path: '/etc/mysql/conf.d/galera.cnf'
  register: check_galeracnf

- name: Copy Galera bootstrap configuration file
  template:
    src: roles/galera/templates/galera_replica.cnf
    dest: /etc/mysql/conf.d/galera.cnf
  register: galeracnf
  when: check_galeracnf.stat.exists == False

- name: Restart mariadb
  systemd:
    name: mariadb.service
    state: restarted
  when: galeracnf.changed
```

> Important c'est semi-automatique, sur notre master il fautdra ajouter les ips du cluster dans "wsrep_cluster_address="gcomm://"

### Web

Ce rôle configure notre serveur web. Pour cela, on installe les packages requis, puis on récupère la configuration de Kanboard depuis notre serveur NFS. La configuration du vhost est ensuite réalisée, incluant la désactivation du site par défaut d'Apache (`000-default.conf`) s'il est activé, suivi de la copie de la configuration du site Kanboard (`kanboard.conf`). Enfin, le playbook vérifie la présence du fichier de configuration de Kanboard (`config.php`) et le duplique s'il n'est pas déjà présent.

```yml
---
- name: Install apache2 and php dependencies
  apt:
    update_cache: yes
    pkg:
      - apache2
      - libapache2-mod-php8.2
      - php8.2
      - php8.2-mysql
      - php8.2-gd
      - php8.2-mbstring
      - php8.2-imagick
      - php8.2-xml
      - nfs-common
    state: latest

- name: Restart apache2
  systemd:
    name: apache2.service
    state: restarted

- name: Add /var/www/nfs path
  file:
    path: /var/www/nfs
    state: directory
    mode: '0755'
    owner: www-data
    group: www-data

- name: Add nfs mount
  lineinfile:
    path: /etc/fstab
    regexp: '^.*/srv/nfs.*$'
    line: '10.10.61.20:/srv/nfs /var/www/nfs nfs noatime,nodiratime 0 0'
    state: present

- name: Mount nfs share
  command: mount -a

- name: Chown export to www-data:www-data
  command: chown -R www-data:www-data /var/www/nfs

- name: Check if default vhost is disable
  stat:
    path: '/etc/apache2/sites-enabled/000-default.conf'
  register: check_defaultvhost

- name: Disable default apache2 site
  command: a2dissite 000-default
  when: check_defaultvhost.stat.exists == True

- name: Check if kanboard vhost exist
  stat:
    path: '/etc/apache2/sites-available/kanboard.conf'
  register: check_vhostkanboard

- name: Copy vhost
  template:
    src: roles/web/templates/kanboard.conf
    dest: /etc/apache2/sites-available/kanboard.conf
  when: check_vhostkanboard.stat.exists == False

- name: Check if kanboard vhost link exists
  stat:
    path: '/etc/apache2/sites-enabled/kanboard.conf'
  register: check_linkkanboard

- name: Enable kanboard apache2 site
  command: a2ensite kanboard
  when: check_linkkanboard.stat.exists == False

- name: Check if kanboard config.php exists
  stat:
    path: '/var/www/nfs/kanboard-1.2.34/config.php'
  register: check_configkanboard

- name: Copy config.php
  template:
    src: roles/web/templates/config.php
    dest: /var/www/nfs/kanboard-1.2.34/config.php
  when: check_configkanboard.stat.exists == False

- name: Restart apache2
  systemd:
    name: apache2.service
    state: restarted
  when: check_linkkanboard.stat.exists == False
```
### NFS

Ce playbook a pour objectif de configurer un serveur NFS avec des volumes logiques. Il installe les paquets nécessaires, vérifie si le volume `/dev/sdb` est initialisé, et crée le volume physique, le groupe de volumes, et le volume logique si nécessaire. Il crée ensuite un système de fichiers, monte le volume dans `/srv/nfs`, exporte le répertoire à un sous-réseau spécifique, et vérifie l'existence du répertoire Kanboard avant de le télécharger si nécessaire.

```yml
---
- name: Install nfs server and LVM packages
  apt:
    update_cache: yes
    pkg:
      - nfs-kernel-server
      - lvm2
    state: latest

- name: Check if /dev/sdb is initialized
  command: pvdisplay /dev/sdb
  register: pvdisplay_output
  ignore_errors: yes

- name: Create physical volume
  command: pvcreate /dev/sdb
  when: "pvdisplay_output.rc != 0"

- name: Create volume group
  command: vgcreate vg_nfs /dev/sdb
  when: "pvdisplay_output.rc != 0"

- name: Create logical volume
  command: lvcreate -l 100%FREE -n lv_nfs vg_nfs
  when: "pvdisplay_output.rc != 0"

- name: Create filesystem on logical volume
  filesystem:
    fstype: ext4
    dev: /dev/vg_nfs/lv_nfs
  when: "pvdisplay_output.rc != 0"

- name: Create directory /srv/nfs
  file:
    path: /srv/nfs
    state: directory
    mode: '0755'
  when: "pvdisplay_output.rc != 0"

- name: Mount logical volume in /srv/nfs
  mount:
    path: /srv/nfs
    src: /dev/vg_nfs/lv_nfs
    fstype: ext4
    state: mounted

- name: Export directory to subnet front / back
  lineinfile:
    path: /etc/exports
    regex: '^/srv/nfs.*$'
    line: '/srv/nfs 10.10.60.0/24(rw,sync,no_subtree_check,no_root_squash,anonuid=33,anongid=33)'

- name: Export filesystems
  command: exportfs -a

- name: Check Kanboard directory existence
  stat:
    path: /srv/nfs/kanboard
  register: kanboardinstall

- name: Download Kanboard archive
  unarchive:
    src: https://github.com/kanboard/kanboard/archive/refs/tags/v1.2.34.tar.gz
    dest: /srv/nfs/
    remote_src: yes
  when: not kanboardinstall.stat.exists
```
### Loki
On a choisi les solutions Grafana, Loki, et Promtail car elles ont été créées par Grafana Labs. Ensemble, elles sont plus optimisées.
La stack "Graylog" est beacoup trop lourde à maintenir et à utiliser + configurer. On part sur de la simplicité avec Loki + Grafana. Il suffit juste d'installer le binaire.

```yml
---
- name: Installation unzip
  ansible.builtin.package:
    name:
      - unzip
    state: present

- name: Telecharger Loki
  get_url:
    url: "https://github.com/grafana/loki/releases/download/v2.8.7/loki-linux-amd64.zip"
    dest: /tmp/loki-linux-amd64.zip
  register: download

- name: Telecharge & decompresse loki-linux-amd64.zip
  ansible.builtin.unarchive:
    src: https://github.com/grafana/loki/releases/download/v2.8.7/loki-linux-amd64.zip
    dest: /usr/local/bin/
    remote_src: yes

- name: Ajout droit +x sur /usr/local/bin/loki-linux-amd64
  file:
    path: /usr/local/bin/loki-linux-amd64
    state: file
    owner: root
    group: root
    mode: 0755

- name: Creation dossier /etc/loki
  ansible.builtin.file:
    path: /etc/loki
    state: directory

- name: Creation dossier /var/lib/loki
  ansible.builtin.file:
    path: /var/lib/loki
    state: directory

- name: Verification si configuration loki.yml existe
  stat:
    path: '/etc/loki/loki.yml'
  register: check_loki

- name: Copie de la confiuration loki.yml
  template:
    src: roles/loki/templates/loki.yml
    dest: /etc/loki/loki.yml
  when: check_loki.stat.exists == False

- name: Verification si configuration si une configuration systemd loki.service existe
  stat:
    path: '/etc/systemd/system/loki.service'
  register: check_loki_systemd

- name: Copie de la configuration systemd
  template:
    src: roles/loki/templates/loki.service
    dest: /etc/systemd/system/loki.service
  when: check_loki_systemd.stat.exists == False

- name: Recharge systemd daemons
  systemd:
    daemon_reload: yes

- name: Lancement Loki
  systemd:
    name: loki
    state: started

- name: Activation de loki.service
  systemd:
    name: loki
    enabled: yes

```

### Promtail

Sur l'ensemble des machines à superviser on installe l'agent promtail qui scrappe les logs et l'envoie à loki.

```yml
---
- name: Installation unzip
  ansible.builtin.package:
    name:
      - unzip
    state: present
    
- name: Telecharge & decompresse loki-linux-amd64.zip
  ansible.builtin.unarchive:
    src: https://github.com/grafana/loki/releases/download/v2.8.7/promtail-linux-amd64.zip
    dest: /usr/local/bin/
    remote_src: yes

- name: Ajout droit +x sur /usr/local/bin/promtail-linux-amd64
  file:
    path: /usr/local/bin/promtail-linux-amd64
    state: file
    owner: root
    group: root
    mode: 0755

- name: Creation dossier /etc/promtail
  ansible.builtin.file:
    path: /etc/promtail
    state: directory

- name: Vérification si configuration promtail.yml existe
  stat:
    path: '/etc/promtail/promtail.yml'
  register: check_promtail

- name: Copie de la configuration promtail.yml
  template:
    src: "roles/promtail/templates/promtail.yml.j2"
    dest: '/etc/promtail/promtail.yml'
  when: check_promtail.stat.exists == False

- name: Verification si une configuration systemd promtail.service existe
  stat:
    path: '/etc/systemd/system/promtail.service'
  register: check_promtail_systemd

- name: Copie de la configuration systemd
  template:
    src: roles/promtail/templates/promtail.service
    dest: /etc/systemd/system/promtail.service
  when: check_promtail_systemd.stat.exists == False

- name: Recharge systemd daemons
  systemd:
    daemon_reload: yes

- name: Lancement Promtail
  systemd:
    name: promtail
    state: started

- name: Activation Promtail
  systemd:
    name: promtail
    enabled: yes
```

Dans notre template de configuration scrapping  on exploite des variables Jinja2 pour faire des fichiers configurations dynamiques.
```jinja2
  url: http://{{ hostvars[groups['loki'][0]].ansible_host }}:3100/loki/api/v1/push
```

### Grafana

Grafana va exploiter les logs dans loki. On reste sur l'ecosysteme Grafana Labs

```yml
---
- name: Installation pre-requis pour grafana-server
  apt:
    update_cache: yes
    pkg:
      - apt-transport-https
      - software-properties-common
      - wget

- name: Creation du dossier pour les cles GPG
  ansible.builtin.file:
    path: /etc/apt/keyrings/
    state: directory

- name: Ajout de la cle GPG Grafana
  apt_key:
    url: https://apt.grafana.com/gpg.key
    state: present

- name: Ajout du depot Grafana
  ansible.builtin.apt_repository:
    repo: 'deb https://apt.grafana.com stable main'
    state: present
    update_cache: yes

- name: Installation Grafana OSS
  apt:
    update_cache: yes
    pkg:
      - grafana

- name: Activation et lancement grafana-server
  systemd:
    name: grafana-server
    state: started
```
## Démonstration

Voici des courtes vidéos démontrant l'automatisation & la haute disponibilité des services.

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

### Configuration YAML Promtail sur le serveur Apache-00 (C'est le même sur Apache-01) :

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

        ### Serveur :

        http_listen_port : C'est le port sur lequel Promtail va écouter pour le trafic HTTP.
        grpc_listen_port : C'est pour le gRPC, mais ici c'est mis à zéro donc on ne l'utilise pas.

        ### Configuration du client :
        Url : C'est l'adresse de notre instance Loki où Promtail va envoyer les logs.

        ### Configurations de récupération (scraping) :
        job_name : On donne un nom à notre job pour pouvoir le repérer plus facilement.
        static_configs : Ici, on définit des cibles statiques pour notre job, avec des labels spécifiques.
        journal : Ça c'est pour récupérer les logs du journal de systemd.
        path : Le chemin vers les fichiers de log qu'on veut récupérer.
        relabel_configs : Ça nous permet de modifier les labels des logs dynamiquement.

        ### Étapes de traitement (pipeline stages) :
        match : Cette étape filtre les logs en utilisant le sélecteur fourni.
        regex : On utilise des expressions régulières pour extraire des champs spécifiques des logs.
        labels : Les champs extraits sont transformés en labels pour le flux de logs.
        Dans notre cas, on extrait pas mal d'infos des logs Apache, comme l'IP du client, la méthode de requête, le chemin, la version HTTP, le code de réponse, la taille de la réponse, le référent et l'agent utilisateur.

        Cette configuration nous permet de faire des requêtes sur nos logs dans Loki en utilisant ces labels, ce qui facilite grandement le filtrage et la recherche dans nos données de logs. Par exemple, on peut chercher tous les logs qui ont un code de réponse spécifique, ou voir toutes les requêtes faites par une certaine IP client.

        
        ## Commande pour generer les logs avec ApacheBench:
   
        ### c'est un outil de test de charge pour les serveurs web connu sous le nom d'ApacheBench. Il est utilisé pour effectuer des tests de performance sur des serveurs HTTP.

        ### ab -n 60 -c 30 http://10.10.60.20/

          ab : C'est le programme ApacheBench lui-même.
          -n 60 : Cela indique à ApacheBench de faire un total de 60 requêtes HTTP pendant le test.
          -c 30 : Cela définit la concurrence à 30, ce qui signifie que ApacheBench va essayer de faire 30 requêtes en parallèle (simultanément) jusqu'à ce que le total de 60 requêtes soit atteint.
          http://10.10.60.20/ : C'est l'URL cible du test de charge. Ici c'est notre Apache-00.

