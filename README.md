# TP 23 : Migration de Eureka vers Consul

## Description du Projet

Ce projet démontre la migration d'une architecture microservices d'Eureka vers Consul pour la découverte de services. Il comprend plusieurs microservices Spring Boot qui utilisent Consul pour l'enregistrement et la découverte automatique des services.

## Architecture

L'architecture comprend les microservices suivants :

- **Car Service** (`car/`) : Service de gestion des voitures avec Spring Data JPA et H2 database
- **Client Service** (`client/`) : Service client avec Spring Data JPA et H2 database
- **Gateway Service** (`gateway/`) : API Gateway utilisant Spring Cloud Gateway pour router les requêtes
- **Server Eureka** (`server_eureka/`) : Service de registre (maintenant configuré avec Consul au lieu d'Eureka)

## Technologies Utilisées

- **Spring Boot 3.2.0** : Framework principal
- **Spring Cloud 2023.0.0** : Pour la découverte de services
- **Consul** : Service discovery et configuration
- **Spring Cloud Gateway** : API Gateway
- **Spring Data JPA** : Persistence des données
- **H2 Database** : Base de données en mémoire
- **Maven** : Gestion des dépendances

## Prérequis

- Java 17+
- Maven 3.6+
- Docker et Docker Compose (pour déploiement conteneurisé)
- Git

## Installation et Configuration

### 1. Cloner le Repository

```bash
git clone https://github.com/houssamb4/microservices-consul-migration.git
cd microservices-consul-migration
```

### 2. Démarrer Consul

#### Option 1 : Via Docker (Recommandé)

```bash
docker run -d --name consul -p 8500:8500 -p 8600:8600/udp consul:latest agent -server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0
```

#### Option 2 : Via Executable

Télécharger Consul depuis https://www.consul.io/downloads et exécuter :

```bash
consul agent -server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0
```

### 3. Configurer les Services

Chaque service utilise H2 comme base de données en mémoire. Les configurations sont dans `application.yml` :

- **Car Service** : Port 8082, base H2 `carservicedb`
- **Client Service** : Port 8081, base H2 `clientservicedb`
- **Gateway Service** : Port 8888
- **Server Eureka** : Port 8761 (mais configuré pour Consul)

## Démarrage des Services

### Démarrer les Microservices

Ouvrir des terminaux séparés pour chaque service :

```bash
# Terminal 1 : Car Service
cd car
mvn spring-boot:run

# Terminal 2 : Client Service
cd ../client
mvn spring-boot:run

# Terminal 3 : Gateway Service
cd ../gateway
mvn spring-boot:run

# Terminal 4 : Server Eureka (optionnel, pour compatibilité)
cd ../server_eureka
mvn spring-boot:run
```

### Ordre de Démarrage Recommandé

1. Consul
2. Car Service
3. Client Service
4. Gateway Service

## Vérification du Fonctionnement

### Interface Web de Consul

Accéder à l'interface de Consul : http://localhost:8500/

Vous devriez voir :
- Services enregistrés : `SERVICE-CAR`, `SERVICE-CLIENT`, `Gateway`
- État de santé : Tous en `passing`

### Tests des Endpoints

#### Via Gateway

```bash
# Car Service via Gateway
curl http://localhost:8888/car

# Client Service via Gateway
curl http://localhost:8888/client
```

#### Directement

```bash
# Car Service
curl http://localhost:8082/actuator/health

# Client Service
curl http://localhost:8081/actuator/health

# Gateway
curl http://localhost:8888/actuator/health
```

### Health Checks

Tous les services exposent des endpoints de santé :

- `/actuator/health` : État général
- `/actuator/info` : Informations du service

## Configuration Consul

Chaque service est configuré pour :

- S'enregistrer automatiquement dans Consul
- Effectuer des health checks HTTP toutes les 10 secondes
- Utiliser le port configuré pour les checks

Exemple de configuration dans `application.yml` :

```yaml
spring:
  cloud:
    consul:
      discovery:
        enabled: true
        register: true
        health-check-path: /actuator/health
        health-check-interval: 10s
```

## Dépannage

### Services Non Enregistrés

- Vérifier que Consul fonctionne : `curl http://localhost:8500/v1/status/leader`
- Vérifier les logs des services pour les erreurs de connexion
- S'assurer que les ports ne sont pas utilisés

### Health Checks Failing

- Vérifier que l'endpoint `/actuator/health` répond : `curl http://localhost:PORT/actuator/health`
- Vérifier la configuration des health checks dans `application.yml`

### Problèmes de Découverte

- Désactiver temporairement le load balancer si nécessaire : `spring.cloud.loadbalancer.enabled=false`
- Vérifier la configuration Gateway pour la découverte automatique

## Migration Eureka → Consul

### Différences Clés

| Aspect | Eureka | Consul |
|--------|--------|--------|
| Protocole | HTTP/REST | HTTP + Gossip |
| Stockage | In-memory | Consensus (Raft) |
| HA | Replication | Multi-datacenter |
| UI | Basique | Avancée |
| Configuration | Spring Cloud Netflix | Spring Cloud Consul |

### Étapes de Migration

1. ✅ Remplacer `spring-cloud-starter-netflix-eureka-client` par `spring-cloud-starter-consul-discovery`
2. ✅ Configurer les propriétés Consul dans `application.yml`
3. ✅ Supprimer les annotations `@EnableEurekaClient`
4. ✅ Adapter la configuration Gateway
5. ✅ Tester l'enregistrement et la découverte

