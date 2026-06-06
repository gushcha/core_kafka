# Kafka — core_kafka

This repository contains the Kubernetes manifests to run a single-broker Apache Kafka instance
on the shared k3s cluster, used as the message bus between `core_api` and `interaction_api`.

---

## Why Kafka?

Kafka decouples services so they don't call each other directly. Instead of `interaction_api`
making an HTTP request to `core_api` and waiting for a weather response inside the WebSocket
handler, it publishes a message and continues. `core_api` picks it up whenever it's ready
and publishes the result back. Each service evolves and scales independently.

Reference: [Kafka in a nutshell](https://kafka.apache.org/intro)

---

## Architecture

```
browser
  │  WebSocket
interaction_api ──► weather.requests ──► core_api
interaction_api ◄── weather.results  ◄── core_api
                                              │
                                         wttr.in API
```

### Topics

| Topic              | Producer         | Consumer         | Purpose                        |
|--------------------|------------------|------------------|--------------------------------|
| `weather.requests` | interaction_api  | core_api         | Client asked for weather       |
| `weather.results`  | core_api         | interaction_api  | Weather data ready to deliver  |

### Envelope format

Every message on both topics uses the same JSON envelope:

```json
{
  "action":    "get_weather" | "weather_result",
  "client_id": "<uuid>",
  "payload":   null | { ...weather data... }
}
```

`client_id` is a UUID assigned to each WebSocket connection by `interaction_api`.
It travels through Kafka so `interaction_api` knows which client to forward the result to.

---

## Setup — KRaft mode (no Zookeeper)

This deployment uses **KRaft** (Kafka Raft metadata mode), available since Kafka 2.8 and
the default since 3.3. It removes the Zookeeper dependency, making the cluster simpler to
operate in Kubernetes.

Reference: [KRaft overview](https://kafka.apache.org/documentation/#kraft)

### Why 1 broker for this project

A production Kafka cluster needs at least **3 brokers** to form a proper KRaft quorum
(majority vote requires ⌊n/2⌋+1 nodes). With 2 brokers, losing one loses quorum.
With 1 broker, there is no replication but also no quorum complexity — perfectly fine
for a study project where data durability is not a concern.

For a production setup, start with the [Strimzi operator](https://strimzi.io/), which
manages Kafka on Kubernetes with proper replication, rolling upgrades, and TLS.

---

## Deploy

```bash
# On the control plane
git clone https://github.com/gushcha/core_kafka.git /opt/core_kafka
cd /opt/core_kafka

kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/kafka.yaml

# Verify — the pod takes ~30s to become ready
kubectl get pods -n kafka -w
```

Once running, the broker is reachable inside the cluster at:

```
kafka.kafka.svc.cluster.local:9092
```

### Verify topics are created automatically

```bash
kubectl exec -it kafka-0 -n kafka -- \
  kafka-topics.sh --bootstrap-server localhost:9092 --list
```

After the first message flows through, you should see `weather.requests` and `weather.results`.

---

## Further reading

- [Kafka documentation](https://kafka.apache.org/documentation/)
- [rdkafka Rust crate](https://docs.rs/rdkafka) — librdkafka bindings used by both services
- [Strimzi — Kafka on Kubernetes](https://strimzi.io/) — production-grade alternative
- [Kafka consumer groups explained](https://www.confluent.io/blog/kafka-consumer-group-protocol/)
- [KRaft: Kafka without Zookeeper](https://developer.confluent.io/learn/kraft/)
