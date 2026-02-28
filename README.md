# Kafka Server — Infraestructura de mensajería

Cluster de Apache Kafka con 3 brokers y 3 nodos Zookeeper levantado con Docker Compose. Incluye una interfaz web para monitoreo y gestión de topics.

---

## Servicios

| Servicio       | Imagen                          | Puerto host | Puerto interno | Descripción                        |
|----------------|---------------------------------|-------------|----------------|------------------------------------|
| `zookeeper-1`  | confluentinc/cp-zookeeper:7.4.4 | 22181       | 2181           | Coordinación del cluster (nodo 1)  |
| `zookeeper-2`  | confluentinc/cp-zookeeper:7.4.4 | 32181       | 2181           | Coordinación del cluster (nodo 2)  |
| `zookeeper-3`  | confluentinc/cp-zookeeper:7.4.4 | 42181       | 2181           | Coordinación del cluster (nodo 3)  |
| `kafka-1`      | confluentinc/cp-kafka:7.4.4     | 29092       | 9092           | Broker Kafka (ID: 1)               |
| `kafka-2`      | confluentinc/cp-kafka:7.4.4     | 39092       | 9092           | Broker Kafka (ID: 2)               |
| `kafka-3`      | confluentinc/cp-kafka:7.4.4     | 49092       | 9092           | Broker Kafka (ID: 3)               |
| `kafka-ui`     | provectuslabs/kafka-ui:latest   | 8090        | 8080           | Interfaz web de administración     |

---

## Levantar el cluster

```bash
docker compose up -d
```

El orden de inicio es gestionado automáticamente mediante `healthcheck` y `depends_on`:

```
zookeeper-1/2/3  →  kafka-1/2/3  →  kafka-ui
```

Espera aproximadamente **60–90 segundos** a que todos los servicios pasen a estado `healthy` antes de conectar los microservicios.

## Detener el cluster

```bash
# Detener sin borrar datos
docker compose down

# Detener y eliminar volúmenes (borra todos los datos y mensajes)
docker compose down -v
```

---

## Configuración del cluster

### Listeners (red dual)

Cada broker expone **dos listeners** para separar el tráfico interno del externo:

| Listener        | Uso                                              | Dirección interna   |
|-----------------|--------------------------------------------------|---------------------|
| `PLAINTEXT`     | Comunicación entre brokers y microservicios      | `kafka-N:9092`      |
| `PLAINTEXT_HOST`| Acceso desde el host (herramientas locales)      | `localhost:X9092`   |

Los microservicios dentro de Docker deben usar `kafka-1:9092,kafka-2:9092,kafka-3:9092`.
Las herramientas en el host deben usar `localhost:29092,localhost:39092,localhost:49092`.

### Replicación y consistencia

| Parámetro                                   | Valor | Descripción                                      |
|---------------------------------------------|-------|--------------------------------------------------|
| `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR`    | 2     | Réplicas del topic interno de offsets            |
| `KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR` | 2 | Réplicas del log de transacciones             |
| `KAFKA_MIN_INSYNC_REPLICAS`                 | 2     | Mínimo de réplicas sincronizadas para escribir   |
| `KAFKA_AUTO_CREATE_TOPICS_ENABLE`           | false | Los topics deben crearse explícitamente          |

### Volúmenes persistentes

```
zk1-data, zk2-data, zk3-data     → datos de Zookeeper
kafka1-data, kafka2-data, kafka3-data  → datos y logs de Kafka
```

### Red Docker

Todos los servicios comparten la red `kafka-net` (bridge). Los microservicios que necesiten conectarse al cluster deben unirse a esta misma red.

---

## Kafka UI

Interfaz web para explorar topics, mensajes, grupos de consumidores y el estado del cluster.

**URL:** [http://localhost:8090](http://localhost:8090)

Cluster configurado: `kafka-local` → `kafka-1:9092, kafka-2:9092, kafka-3:9092`

---

## Topics del proyecto

Los topics **no se crean automáticamente** (`AUTO_CREATE_TOPICS_ENABLE=false`). Son creados por los microservicios al iniciar mediante beans `NewTopic`.

| Topic                   | Particiones | Réplicas | Creado por                    |
|-------------------------|-------------|----------|-------------------------------|
| `ubicaciones_vehiculos` | 3           | 2        | `productor-ubicaciones`       |
| `horarios`              | 3           | 2        | `consumidor-procesa-senales`  |

---

## Verificar estado del cluster

```bash
# Ver estado de los contenedores
docker compose ps

# Ver logs de un broker
docker logs kafka-1

# Listar topics desde el host
docker exec kafka-1 kafka-topics --bootstrap-server localhost:9092 --list
```
