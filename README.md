# Event Driven Architecture

Cette activité se focalise sur la mise en place de l'Event Driven Architecture en utilisant Apache Kafka et Spring Cloud
Streams. L'EDA est un modèle de conception qui permet de créer des systèmes réactifs en temps réel en réagissant aux
événements.

Nous aborderons les étapes clés, de l'installation de Kafka à l'utilisation de Docker pour simplifier le déploiement. De
plus, nous développerons divers services Kafka, tels qu'un producteur, un consommateur, un fournisseur et un service de
traitement en temps réel.

L'objectif final est la création d'une application web pour visualiser en temps réel les résultats du traitement des
données.

## Table de matière

- [Démarrer Zookeeper](#Démarrer Zookeeper)
- [Démarrer Kafka-server](#Démarrer Kafka-server)
- [Tester avec Kafka-console-producer et kafka-console-consumer](#Tester avec Kafka-console-producer et kafka-console-consumer)
- [Avec Docker](#Avec Docker)
- [Avec KAFKA & Spring Cloud Streams](#AvecKAFKA&StpringCloudStreams)
  
![Architecture](.jpg)
## Démarrer Zookeeper

Après le téléchargement du Kafka on va démarrer zookeeper qui est utilisé comme composant de base pour la gestion des brokers Kafka

Pour le démarrer, on va exécuter la commande suivante :

```shell
start bin\windows\zookeeper-server-start.bat config/zookeeper.properties
```
![zookeeper](.jpg)

Zookeeper, maintenant, est démarré sur le port 2181 :

## Démarrer Kafka-server

On va démarrer Kafka-server à l’aide de cette commande :
```shell
start bin\windows\kafka-server-start.bat config/server.properties
```
Après l'exécution de la commande, Kafka va écouter sur l'adresse IP 0.0.0.0 et sur le port 9092.

## Tester avec Kafka-console-producer et kafka-console-consumer
* kafka-console-consumer

L'utilitaire kafka-console-consumer est utilisé pour consommer des messages à partir d'un topic Kafka à partir de la
ligne de commande.

On va démarrer notre consommateur de console Kafka sur un système Windows, le configurer pour se connecter à un broker
Kafka sur localhost:9092 et lui indique de lire les messages du topic R1 en éxécutant la commande : 
```shell
start bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic R1
```

* kafka-console-producer

L'utilitaire kafka-console-producer est utilisé pour produire (envoyer) des messages vers un topic Kafka à partir de la
ligne de commande.

Comme la commande montre, ce producteur doit se connecter au brokers Kafka fonctionnant sur localhost sur le port 9092
pour envoyer des messages au topic nommé R1
<br/><br/>
```shell
start bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic R1
```

Alors le consommateur attend les messages envoyés vers le topic <br/><br/>
![image](.jpg)

## Avec Docker

* Créer le fichier docker-compose.yml

![image](.jpg)

* Démarrer les conteneurs docker : zookeeper et kafka-broker

![image](.jpg)

* Tester avec Kafka-console-producer et kafka-console-consumer

![image](.jpg)

![image](.jpg)

## Avec KAFKA & Stpring Cloud Streams

* Un Service Producer KAFKA via un RestController

  ```java
  @GetMapping("/publish/{topic}/{name}")
  public PageEvent publish(@PathVariable String topic, @PathVariable String name) {
  PageEvent pageEvent = new PageEvent(name, Math.random() > 0.5 ? "U1" : "U2", new Date(), new Random().nextInt(9000));
  streamBridge.send(topic, pageEvent);
  return pageEvent;
  }
  
* Un Service Consumer KAFKA
  <br><br>
  Le rôle de ce service est de consommer des événements de type PageEvent et d'afficher ces événements dans la console.
  Il sert principalement à observer les événements reçus dans le système et à fournir une visibilité sur les données qui
  transitent par le flux de données Kafka.

  ```java
    @Bean
    public Consumer<PageEvent> pageEventConsumer() {
        return (input) -> {
            System.out.println("*************************");
            System.out.println(input.toString());
            System.out.println("*************************");
        };
    }
  
* Un Service Supplier KAFKA
  <br><br>
  Nous avons créé un Supplier de PageEvent qui est une interface fonctionnelle de Java qui ne prend aucun argument et
  renvoie un résultat. Il génère aléatoirement des objets PageEvent à chaque appel.

  ```java
    @Bean
    public Supplier<PageEvent> pageEventSupplier() {
        return () -> new PageEvent(
                Math.random() < 0.5 ? "P1" : "P2",
                Math.random() > 0.5 ? "U1" : "U2",
                new Date(),
                new Random().nextInt(9000));
    }
* Un Service Function KAFKA
  ```java
    @Bean
    public Function<PageEvent, PageEvent> pageEventFunction() {
        return (input) -> {
            input.setName("Page event");
            input.setUser("user");
            return input;
        };
    }
  
* Un Service de Data Analytics Real Time Stream Processing avec Kafka Streams
  Un service de Data Analytics en temps réel basé sur Kafka Streams est un système qui permet de traiter, analyser et
  transformer des données en continu à mesure qu'elles sont générées. Kafka Streams est une bibliothèque open-source
  pour le traitement des flux de données en temps réel, conçue pour être intégrée avec Apache Kafka, une plateforme de
  streaming de données.

  ```java
    @Bean
    public Function<KStream<String, PageEvent>, KStream<String, Long>> kStreamFunction() {
        return (input) -> input
                .filter((k, v) -> v.getDuration() > 100)
                .map((k, v) -> new KeyValue<>(v.getName(), 0L))
                .groupBy((k, v) -> k, Grouped.with(Serdes.String(), Serdes.Long()))
                .windowedBy(TimeWindows.of(Duration.ofSeconds(5)))
                .count(Materialized.as("page-count"))
                .toStream()
                .map((k, v) -> new KeyValue<>("=>" + k.window().startTime() + k.window().endTime() + k.key(), v));
    }


On va ajouter ces propriétés :

  ```properties
  spring.cloud.stream.bindings.kStreamFunction-in-0.destination=R2
  spring.cloud.stream.bindings.kStreamFunction-out-0.destination=R4
  spring.cloud.stream.kafka.streams.binder.configuration.commit.interval.ms=1000
  ```
* On va exécuter la commande suivante pour démarrer un consommateur Kafka en mode console, le configure pour se connecter
à un serveur Kafka local, lire les messages du sujet "R4", et afficher à la fois les clés et les valeurs des messages,
avec la configuration spécifiée pour les désérialiseurs de clé et de valeur.

  ```shell
  start bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic R4 --property print.value=true
  --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer --property
  value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
  ```
* Une application Web qui permet d'afficher les résultats du Stream Data Analytics en temps réel
  On va exposer (/analytics) pour fournir un flux continu de données d'analyse en temps réel au format Server-Sent
  Events

  ```java
    @GetMapping(value = "/analytics", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Map<String, Long>> analytics() {
        return Flux.interval(Duration.ofSeconds(1))
                .map(seq -> {
                    Map<String, Long> stringLongMap = new HashMap<>();
                    ReadOnlyWindowStore<String, Long> windowStore = interactiveQueryService.getQueryableStore("page-count",
                            QueryableStoreTypes.windowStore());
                    Instant now = Instant.now();
                    Instant from = now.minusMillis(5000);
                    KeyValueIterator<Windowed<String>, Long> fetchAll = windowStore.fetchAll(from, now);
                    while (fetchAll.hasNext()) {
                        KeyValue<Windowed<String>, Long> next = fetchAll.next();
                        stringLongMap.put(next.key.key(), next.value);
                    }
                    return stringLongMap;
                }).share();
    }
  
On va utiliser Smoothie une bibliothèque JavaScript populaire pour la création de graphiques en temps réel et
d'animations de manière fluide dans le navigateur web

```javascript

<script>
    var index = -1;
    randomColor = function () {
        ++index;
        if (index >= colors.length) index = 0;
        return colors[index];
    }
    var pages = ["P1", "P2"];
    var colors = [
        {sroke: 'rgba(0, 255, 0, 1)', fill: 'rgba(0, 255, 0, 0.2)'},
        {
            sroke: 'rgba(255, 0, 0, 1)', fill: 'rgba(255, 0, 0, 0.2)'
        }];
    var courbe = [];
    var smoothieChart = new SmoothieChart({tooltip: true});
    smoothieChart.streamTo(document.getElementById("chart2"), 500);
    pages.forEach(function (v) {
        courbe[v] = new TimeSeries();
        col = randomColor();
        smoothieChart.addTimeSeries(courbe[v], {strokeStyle: col.sroke, fillStyle: col.fill, lineWidth: 2});
    });
    var stockEventSource = new EventSource("/analytics");
    stockEventSource.addEventListener("message", function (event) {
        pages.forEach(function (v) {
            val = JSON.parse(event.data)[v];
            courbe[v].append(new Date().getTime(), val);
        });
    });
</script>
```
Nous avons ajoué ce script pour créer un graphique en temps réel en utilisant la bibliothèque SmoothieChart. 

Il récupère des données d'événements provenant d'une source externe via EventSource et les affiche graphiquement en mettant à jour le graphique en continu. Cela permet de visualiser en temps réel les données relatives à des pages spécifiques (P1 et P2) dans des couleurs différentes.
## 🔗 About me :

[![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/chaimae-douhi/)
