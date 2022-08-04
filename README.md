# theses-es-cluster-docker

(travail en cours)

Configuration docker 🐳 pour déployer un nœud du cluster elasticsearch de theses.fr (voir aussi le dépôt [``theses-docker``](https://github.com/abes-esr/theses-docker)). 

## Prérequis


- docker
- docker-compose
- réglages ``vm.max_map_count`` pour elasticsearch (cf [FAQ pour les détails du réglage](https://github.com/abes-esr/theses-docker/blob/develop/README-faq.md#comment-r%C3%A9gler-vmmax_map_count-pour-elasticsearch-))

## Installation

Pour intégrer un noeud suplémentaire au cluster elasticsearch de theses.fr, voici la marche à suivre.

On suppose ci-dessous un déploiement de 3 noeuds sur les serveurs suivants (mais on peut extrapoler pour créer 4, 5, ou 6 noeuds si nécessaire) :
- ``diplotaxis1-test``
- ``diplotaxis2-test``
- ``diplotaxis3-test``

Le déploiement d'un nouveau noeud elasticsearch suppose un déploiement préalable de l'application theses.fr (voir [``theses-docker``](https://github.com/abes-esr/theses-docker)) qui embarque en fait le premier noeud elasticsearch. L'installation un peu particulière de ce premier noeud est décrite dans la [première partie de l'installation](#installation--serveur-1--noeud-1).


### Installation : Serveur 1 / Noeud 1

Sur le premier serveur ``diplotaxis1-test`` on va installer le premier noeud elasticsearch en compagnie de toute la pile logicielle de theses.fr. Il y aura donc sur ce premier serveur tous les modules de theses.fr ainsi que le premier noeud du cluster elasticsearch et aussi le kibana d'administration.

Pour installer ce noeud, il faut se reporter à la [section installation du dépôt ``theses-docker``](README.md#installation).

Ensuite sur ce premier noeud, la seule chose à régler concerne les paramètres suivants dans le ``.env`` (ce qui est important c'est de bien choisir le numéro "01" pour le numéro du noeud) :
```env
ELK_CLUSTER_NODE_NUMBER=01
ELK_CLUSTER_DISCOVER_SEED_HOSTS=diplotaxis1-test:10305,diplotaxis2-test:10305,diplotaxis3-test:10305
ELK_CLUSTER_INITIAL_MASTER_NODES=theses-es01,theses-es02,theses-es03
```

Vous devez ensuite lancer l'application classiquement avec cette commande :
```bash
cd /opt/pod/theses-docker/
docker-compose up -d
```

### Installation : Serveurs 2 & 3 / Noeuds 2 & 3

Le second et le troisième noeud elasticsearch de theses.fr sont respectivement déployés sur ``diplotaxis2-test`` et ``diplotaxis3-test`` (cette documentation permet d'extrapoler pour augmenter à 4, 5 ou 6 noeuds elasticsearch si c'était un jour nécessaire). La configuration nécessaire pour déployer ces noeuds suplémentaires est contenu dans ce présent dépôt. Voici les commandes à lancer sur les différents serveurs pour installer les noeuds.

```bash
cd /opt/pod/
git clone https://github.com/abes-esr/theses-es-cluster-docker.git

cd /opt/pod/theses-es-cluster-docker/
mkdir -p volumes/theses-elasticsearch/            && chmod 777 volumes/theses-elasticsearch/
```

Ensuite il faut créer un fichier ``/opt/pod/theses-es-cluster-docker/.env`` en partant du [modèle ``.env-dist``](./.env-dist) :
```bash
cd /opt/pod/theses-es-cluster-docker/
cp .env-dist .env
```

Régler surtout les variables suivantes en prenant soins d'incrémenter le n° du noeud dans ``ELK_CLUSTER_NODE_NUMBER`` sous la forme ``XX`` :
```env
ELK_CLUSTER_NODE_NUMBER=02
ELK_CLUSTER_DISCOVER_SEED_HOSTS=diplotaxis1-test:10305,diplotaxis2-test:10305,diplotaxis3-test:10305
ELK_CLUSTER_INITIAL_MASTER_NODES=theses-es01,theses-es02,theses-es03
```

Et finalement on peut démarrer le noeud elasticsearch qui rejoindra alors le cluster elasticsearch de theses.fr :
```bash
cd /opt/pod/theses-es-cluster-docker/
docker-compose up -d
```

## Supervision

Pour savoir si le cluster à correctement démarré avec ses 3 noeuds, tapez ceci:
```bash
 curl http://diplotaxis1-test:10302/_cluster/health?pretty
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

On peut également observer les logs des noeuds quand un noeud rejoint le cluster :
```json
# coté noeud 1
{
 "@timestamp":"2022-07-27T14:56:55.732Z", "log.level": "INFO",
  "message":"added {{theses-es02}{sH6jSIdMRRyQ5JVowchsyQ}{Sx1tZYh-Rca8nt3GQ2Z-ng}{theses-es02}{172.31.0.2}{172.31.0.2:9300}{cdfhilmrstw}}, term: 5, version: 158, reason: Publication{term=5, version=158}",
  "ecs.version": "1.2.0","service.name":"ES_ECS","event.dataset":"elasticsearch.server","process.thread.name":"elasticsearch[theses-es01][clusterApplierService#updateTask][T#1]",
  "log.logger":"org.elasticsearch.cluster.service.ClusterApplierService",
  "elasticsearch.cluster.uuid":"Y4rayRuGRkasvpxF0DvGTg",
  "elasticsearch.node.id":"I8qD98AIRRyidBDEs_WAqQ",
  "elasticsearch.node.name":"theses-es01",
  "elasticsearch.cluster.name":"theses-cluster"
}

# coté noeud 2
{
  "@timestamp":"2022-07-27T14:56:55.223Z", "log.level": "INFO",
  "message":"master node changed {previous [], current [{theses-es01}{I8qD98AIRRyidBDEs_WAqQ}{8Bc5pUJnTmyG6qNhZDIFTw}{theses-es01}{172.31.0.6}{172.31.0.6:9300}{cdfhilmrstw}]}, added {{theses-es01}{I8qD98AIRRyidBDEs_WAqQ}{8Bc5pUJnTmyG6qNhZDIFTw}{theses-es01}{172.31.0.6}{172.31.0.6:9300}{cdfhilmrstw}}, term: 5, version: 158, reason: ApplyCommitRequest{term=5, version=158, sourceNode={theses-es01}{I8qD98AIRRyidBDEs_WAqQ}{8Bc5pUJnTmyG6qNhZDIFTw}{theses-es01}{172.31.0.6}{172.31.0.6:9300}{cdfhilmrstw}{ml.machine_memory=1073741824, ml.max_jvm_size=536870912, xpack.installed=true}}",
  "ecs.version": "1.2.0","service.name":"ES_ECS","event.dataset":"elasticsearch.server","process.thread.name":"elasticsearch[theses-es02][clusterApplierService#updateTask][T#1]",
  "log.logger":"org.elasticsearch.cluster.service.ClusterApplierService",
  "elasticsearch.node.name":"theses-es02",
  "elasticsearch.cluster.name":"theses-cluster"
}
```


## Mémo

L'authentification ELASTIC_PASSWORD n'est utilisée que pour l'API JSON d'elasticsearch. La communication entre les différents noeuds du cluster elasticsearch est réalisée en se basant sur le système de certificat qui est utilisé via la directive ``xpack.security.transport.ssl.enabled=true``. A noter que dans le cas où la securité est désactivée, la variable ELASTIC_PASSWORD n'est pas utilisée.

Le port 9300 est utilisé (protocole binaire) pour faire communiquer les différents noeuds du cluster elasticsearch (attention à bien l'exposer sur les différents serveurs). Ce n'est pas le port 9200 qui lui est utilisé pour exposer l'API classique d'elasticsearch. Exemple de la configuration où ce port 9300 est utilisé :  
``ELK_CLUSTER_DISCOVER_SEED_HOSTS=theses-elasticsearch-01:9300,theses-elasticsearch-02:9300``
