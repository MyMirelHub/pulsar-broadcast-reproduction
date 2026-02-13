# Pulsar Broadcast & Earliest E2E

This reproduction shows independent broadcast consumption in Dapr with `subscribeInitialPosition: earliest`, including a late-joining consumer.

## Scenario
1. **Continuous Producer**: A Python client (no Dapr) publishing to `topic-broadcast-stable`.
2. **Consumers**: Go apps (Dapr) in separate namespaces (`ns-1`, `ns-2`, `ns-3`, `ns-4`) consuming the stream with unique `consumerID` values.

## Prerequisites
- Pulsar installed in the `pulsar` namespace using the Helm values file at [deploy/pulsar/pulsar-helm-values-minimal.yaml](deploy/pulsar/pulsar-helm-values-minimal.yaml).
- Dapr installed in the cluster.

## Repo Layout
```text
	deploy/
		consumer/
			ns-1.yaml
			ns-2.yaml
			ns-3.yaml
			ns-4.yaml
		producer/
			producer.yaml
		pulsar/
			pulsar-helm-values-minimal.yaml
```

## Config
- Pulsar settings (including 7‑day retention) live in a single Helm values file: [deploy/pulsar/pulsar-helm-values-minimal.yaml](deploy/pulsar/pulsar-helm-values-minimal.yaml).
- Each consumer is defined in one YAML file that includes the namespace, Dapr component, and app deployment: [deploy/consumer/ns-1.yaml](deploy/consumer/ns-1.yaml), [deploy/consumer/ns-2.yaml](deploy/consumer/ns-2.yaml), [deploy/consumer/ns-3.yaml](deploy/consumer/ns-3.yaml), [deploy/consumer/ns-4.yaml](deploy/consumer/ns-4.yaml).
- The producer deployment is in [deploy/producer/producer.yaml](deploy/producer/producer.yaml).

## Install Pulsar
```bash
helm install pulsar apache/pulsar \
	--timeout 10m \
	--namespace pulsar \
	--create-namespace \
	--values deploy/pulsar/pulsar-helm-values-minimal.yaml
```

## Install Dapr
```bash
dapr init -k
```

## Run the E2E

### 1. Start Producer
```bash
kubectl apply -f deploy/producer/producer.yaml
```

### 2. Start Initial Consumers
```bash
kubectl apply -f deploy/consumer/ns-1.yaml
kubectl apply -f deploy/consumer/ns-2.yaml
kubectl apply -f deploy/consumer/ns-3.yaml
```

### 3. Verify "Late Joiner"
Deploy a new consumer (`ns-4`) with a unique `consumerID` and `earliest` configuration. It should replay retained history.

```bash
kubectl apply -f deploy/consumer/ns-4.yaml
```

Check logs for replayed messages:
```bash
kubectl logs -n ns-4 deployments/pulsar-subscriber -f
```

## Notes
- Each namespace has a unique `consumerID` in its Dapr component, enabling independent broadcast.
