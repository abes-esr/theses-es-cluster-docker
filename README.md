# theses-es-cluster-docker

Configuration docker 🐳 pour déployer un nœud du cluster elasticsearch de theses.fr (travail en cours)

## Installation

On commence par récupérer la configuration du déploiement depuis le github :
```bash
cd /opt/pod/ # à adapter en local car vous pouvez cloner le dépôt dans votre homedir
git clone https://github.com/abes-esr/theses-es-cluster-docker.git
```

Ensuite on configure notre déploiement en prenant exemple sur le fichier [``.env-dist``](./.env-dist) qui contient toutes les variables utilisables avec les explications :
```bash
cd /opt/pod/theses-es-cluster-docker/
cp .env-dist .env
# personnalisez alors le .env en partant des valeurs exemple présentes dans le .env-dist
```

Finalement on règle quelques droits sur les répertoires et on peut démarrer le noeud elasticsearch :
```bash
# forcer les droits max pour les volumes déportés sur le système de fichier local
cd /opt/pod/theses-es-cluster-docker/
mkdir -p volumes/theses-elasticsearch/            && chmod 777 volumes/theses-elasticsearch/
mkdir -p volumes/theses-elasticsearch-setupcerts/ && chmod 777 volumes/theses-elasticsearch-setupcerts/
```

Pour déployer theses.fr sur les serveurs de dev, test et prod, il est préférable (obligatoire pour la prod) de passer par un cluster elasticsearch à trois noeuds sur 3 serveurs distincts. Voici la marche à suivre :

On suppose par exemple un déploiement sur les serveurs suivants (remplacer le nom du serveur pour les autres environnements) :
- ``diplotaxis1-test``
- ``diplotaxis2-test``
- ``diplotaxis3-test``

## Serveur 1 : toute l'appli theses.fr + le premier noeud elasticsearch

Sur le premier noeud on va installer la pile logicielle complète de theses.fr qui contient tous les modules de theses.fr ainsi que le premier noeud du cluster elasticsearch et kibana. Pour cela il faut se reporter à la [section installation du dépôt ``theses-docker``](README.md#installation).

Sur ce premier noeud, les réglages particuliers à réaliser dans le ``.env`` sont les suivants :
```env
ELK_CLUSTER_NODE_NUMBER=01
ELK_DISCOVER_SEED_HOSTS=diplotaxis1-test:10305,diplotaxis2-test:10305,diplotaxis3-test:10305
ELK_CLUSTER_INITIAL_MASTER_NODES=theses-es01,theses-es02,theses-es03
```

Vous devez ensuite lancer l'application avec cette commande. Cette opération est indispensable avant de passer à l'étape suivante car elle va générer les certificats nécessaires à la communication inter-cluster (dans ``volumes/theses-elasticsearch-setupcerts/``, cf section suivante) :
```bash
cd /opt/pod/theses-docker/
docker-compose up -d
```

## Serveurs 2 & 3 : les deux autres noeuds elasticsearch de theses.fr

Le second et le troisième noeud elasticsearch de theses.fr sont respectivement déployés sur ``diplotaxis2-test`` et ``diplotaxis2-test``.

```bash
ssh diplotaxis2-test
cd /opt/pod/
git clone https://github.com/abes-esr/theses-es-cluster-docker.git
cd /opt/pod/theses-es-cluster-docker/
mkdir -p volumes/theses-elasticsearch-setupcerts/ && chmod 777 volumes/theses-elasticsearch-setupcerts/
mkdir -p volumes/theses-elasticsearch/ && chmod 777 volumes/theses-elasticsearch/

# Remarque : répéter l'opération sur diplotaxis3-test
```

Vous devez ensuite récupérer les certificats générés par l'initialisation de ``theses-docker`` sur ``diplotaxis1-test`` (cf étape serveur 1 plus haut). Ce sont ces certificats qui permettront aux 3 noeuds elasticsearch de communiquer de façon sécurisée au sein du cluster elasticsearch. Voici comment procéder pour les récupérer et les transmettre aux 2 autres noeuds elasticsearch qui sont sur les serveurs ``diplotaxis2-test`` et ``diplotaxis3-test`` :
```bash
ssh diplotaxis1-test
cd /opt/pod/theses-docker/
docker cp theses-elasticsearch-setupcerts:/usr/share/elasticsearch/config/certs/ca.zip .
docker cp theses-elasticsearch-setupcerts:/usr/share/elasticsearch/config/certs/certs.zip .

scp certs.zip ca.zip diplotaxis2-test:/opt/pod/theses-es-cluster-docker/volumes/theses-elasticsearch-setupcerts/

ssh diplotaxis2-test
cd /opt/pod/theses-es-cluster-docker/volumes/theses-elasticsearch-setupcerts/
unzip certs.zip
unzip ca.zip

# Remarque : répéter l'opération sur diplotaxis3-test
```

Ensuite il faut créer un fichier ``/opt/pod/theses-es-cluster-docker/.env`` en partant du modèle ``.env-dist``.
```bash
cp .env-dist .env

# régler ELASTIC_PASSWORD sur la même valeur que
# dans le .env de theses-docker (diplotaxis1-test)
```

Et finalement on peut démarrer le noeud elasticsearch :
```bash
docker-compose up -d
```

## Supervision

Pour savoir si le cluster à correctement démarré avec ses 3 noeuds, tapez ceci (entrez le mot de passe ``ELASTIC_PASSWORD`` lorsqu'il sera demandé):
```bash
 curl -k -u elastic https://diplotaxis1-test:10302/_cluster/health?pretty
```

Si tout c'est bien passé, vous devriez avoir un retour de ce type :
```json
{
  "cluster_name" : "theses-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 1,
  "active_shards" : 3,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

## Mémo

L'authentification ELASTIC_PASSWORD n'est utilisée que pour l'API JSON d'elasticsearch. La communication entre les différents noeuds du cluster elasticsearch est réalisée en se basant sur le système de certificat qui est utilisé via la directive ``xpack.security.transport.ssl.enabled=true``

Le port 9300 est utilisé (protocole binaire) pour faire communiquer les différents noeuds du cluster elasticsearch (attention à bien l'exposer sur les différents serveurs). Ce n'est pas le port 9200 qui lui est utilisé pour exposer l'API classique d'elasticsearch. Exemple de la configuration où ce port 9300 est utilisé :  
``ELK_DISCOVER_SEED_HOSTS=theses-elasticsearch-01:9300,theses-elasticsearch-02:9300``
