# CFK Connect Pod Logs to Splunk on Kubernetes

End‑to‑end guide for sending **Confluent for Kubernetes (CFK) Kafka Connect pod logs** from **Kubernetes** to **Splunk** using the **Splunk OpenTelemetry Collector for Kubernetes**.

## Target Platform

- **Kubernetes** cluster
- **CFK** already deployed and managing a **Connect** cluster
- `kubectl` configured for the Kubernetes cluster
- `helm` v3.9+ installed

---

## 1. Overview

### Goal

Centralize Kafka Connect logs from CFK‑managed clusters into Splunk, with:

- Kubernetes metadata (cluster, namespace, pod, container)
- Optional JSON formatting for easier parsing
- Minimal impact on existing CFK and Kubernetes setup

### Architecture

```text
+-----------+          +----------------------------+          +-------------------+
| CFK      |  logs    | Splunk OTel Collector      |  HEC     | Splunk            |
| Connect  +--------->+ (DaemonSet on every node)  +--------->+ (Cloud/Enterprise)|
| Pods     | stdout   | - Tails container logs     | HTTPS    | - Index: cfk_*    |
+-----------+          +----------------------------+          +-------------------+
       |                          ^
       |                          |
       v                          |
   Kubernetes / kubelet writes logs to node filesystem
```

---

## 2. Prerequisites

### Platform

- **Kubernetes** cluster
- **CFK** already deployed and managing a **Connect** cluster
- `kubectl` configured for the Kubernetes cluster
- `helm` v3.9+ installed

### Splunk

- Splunk **Cloud** or **Enterprise**
- A **HEC (HTTP Event Collector) endpoint** reachable from Kubernetes:
  - Example: `https://splunk.example.com:8088/services/collector`
- A **HEC token** for Kubernetes logs
- A **log index** for CFK logs, e.g. `cfk_connect_logs`


### Considerations

- The Splunk OTel Collector is a DaemonSet that runs on every node in the cluster.
- The Splunk OTel Collector will tail all container logs by default and send to Splunk via HEC.
- The Splunk OTel Collector will use the `splunkPlatform.endpoint` and `splunkPlatform.token` to authenticate to the Splunk HEC endpoint.
- The Splunk OTel Collector will use the `splunkPlatform.index` to send the logs to the Splunk index.
- The Splunk OTel Collector will use the `splunkPlatform.insecureSkipVerify` to skip certificate verification if the Splunk HEC endpoint is using a self-signed certificate.
- Consider using Index type 'json' or '_json' for the index to automatically parse the logs into fields.
- Configure Confluent's CRs to emit JSON logs that are easier to parse in Splunk via Log4j2 configuration overrides.
- A basic confluent platform deployment with Connect, Kafka, KsqlDB, and Control Center is available in `confluent-platform-c3++.yaml`, with Connect configured to emit JSON logs.
---

## 3. Splunk Setup

You can use either an existing Splunk Cloud/Enterprise deployment or spin up a **free Splunk Cloud Platform trial**.

### 3.1 Create a free Splunk Cloud trial (optional but easy for demos)

1. Go to the **Splunk Cloud Platform trial** page: https://www.splunk.com/en_us/download/splunk-cloud.html
2. Sign up (no credit card required) and create your trial instance (14 days, up to 5 GB/day ingest).
3. **Create index** (if not existing), e.g. `cfk_connect_logs`.
4. When the instance is ready, open **Settings → Data Inputs → HTTP Event Collector (HEC)**.
5. Create a **new HEC token**, note:
   - **Token value** (you’ll paste it into `values.yaml`)
   - **HEC endpoint**, typically `https://<your-instance>.splunkcloud.com:8088/services/collector`

### 3.2 Index and HEC token

For any Splunk deployment (trial or existing):

1. **Create index** (if not existing), e.g. `cfk_connect_logs`.
2. **Create HEC token**:
   - Set default index to `cfk_connect_logs` or rely on index routing later.
3. Note the values for the collector config:
   - **Endpoint**: `https://<hec-host>:8088/services/collector`
   - **Token**: `<hec-token>`

Recommended: one **HEC token per Kubernetes cluster**.

---

## 4. Install Splunk OTel Collector on Kubernetes

### 4.1 Add Helm repo

```bash
helm repo add splunk-otel-collector-chart https://signalfx.github.io/splunk-otel-collector-chart
helm repo update
```

### 4.2 Create a namespace (optional but recommended)

```bash
kubectl create namespace observability
```

### 4.3 Create `splunkValues.yaml`
 
From the template `splunkValuesTemplate.yaml` create `splunkValues.yaml` with the minimum needed config, fill in the values for the Splunk endpoint, token, and index.

### 4.4 Install the chart

```bash
helm -n observability install splunk-otel \
  -f splunkValues.yaml \
  splunk-otel-collector-chart/splunk-otel-collector
```

This deploys:

- **DaemonSet** agents on every node
- Agents tail **all container logs** by default and send to Splunk via HEC

---

## 5. Route Only CFK / Connect Logs

Assumptions:

- CFK is running in namespace `confluent`
- Connect CR is `connect` in that namespace (adjust names as needed)

You can either:

- Route **all CFK logs** to the Splunk index, or
- Route **only Connect pods**.

### 5.1 Option A – All CFK logs from the `confluent` namespace

Annotate the namespace with the desired Splunk index:

```bash
kubectl annotate namespace confluent splunk.com/index=cfk_connect_logs
```

Result:

- All pods in `confluent` namespace have their logs sent to `cfk_connect_logs`.

### 5.2 Option B – Only Connect pods

1. **Exclude the whole namespace by default:**

   ```bash
   kubectl annotate namespace confluent splunk.com/exclude=true
   ```

2. **Include only Connect pods and set index:**

   Annotate the StatefulSet created by CFK to include the Connect pods and set the index.

   ```bash
   kubectl annotate statefulset <statefulset-name> -n confluent \
     splunk.com/include=true \
     splunk.com/index=<index-name>
   ```

---

## 6. (Optional) Make Connect Logs JSON-Friendly

CFK lets you override **Log4j2** configuration for Connect. Use this to emit JSON logs that are easier to parse in Splunk.

In your Connect CR (example):

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: Connect
metadata:
  name: connect
  namespace: confluent
spec:
  configOverrides:
    log4j2:
      Configuration:
        Appenders:
          Console:
            PatternLayout:
              Pattern: '{"severity":"%level","timestamp":"%d{yyyy-MM-dd''T''HH:mm:ss.SSSXXX}","textPayload":"%encode{%X{connector.context}%message%n%ex{full}}{JSON}","sourceLocation":{"file":"%encode{%F}{JSON}","line":"%L","function":"%encode{%c}{JSON}"},"thread":"%encode{%t}{JSON}"}%n'
            name: stdout
            target: SYSTEM_OUT
        Loggers:
          Logger:
          - AppenderRef:
            - ref: stdout
            level: WARN
            name: kafka.authorizer.logger
          Root:
            AppenderRef:
            - ref: stdout
            level: INFO
        name: Log4j2
        status: warn
```

Apply:

```bash
kubectl apply -f connect.yaml
```

After restart/rolling update, Connect pods will emit JSON lines; Splunk can auto-extract fields or you can add field extractions.

---

## 7. Validation

### 7.1 Check OTel agents

```bash
kubectl -n observability get pods -l app=splunk-otel-collector
kubectl -n observability logs -l app=splunk-otel-collector | head
```

Verify there are no repeated HEC errors.

### 7.2 Check Connect pods

```bash
kubectl -n confluent get pods -l "platform.confluent.io/type=connect" -o wide
kubectl -n confluent logs <one-connect-pod> | head
```

Confirm logs are flowing and in the expected format (plain text or JSON).

### 7.3 Verify in Splunk

In Splunk Search:

```spl
index=cfk_connect_logs k8s.cluster.name=Kubernetes-cfk-prod
```

You should see events with fields like:

- `k8s.cluster.name`
- `k8s.namespace.name`
- `k8s.pod.name`
- `k8s.container.name`

To narrow to Connect:

```spl
index=cfk_connect_logs k8s.namespace.name=confluent "org.apache.kafka.connect"
```

---

## 8. Troubleshooting

### No data in Splunk

- Check HEC connectivity:

  ```bash
  kubectl -n observability logs -l app=splunk-otel-collector | grep -i hec
  ```

- Verify:
  - `splunkPlatform.endpoint` URL is correct
  - `splunkPlatform.token` matches HEC token
  - HEC is enabled and accepts connections from Kubernetes IP ranges

### Data in wrong index

- Check pod / namespace annotations:

  ```bash
  kubectl -n confluent get ns confluent -o yaml | grep -i splunk.com
  kubectl -n confluent get pods --show-labels -o yaml | grep -i splunk.com -n
  ```

- Remember:
  - Pod annotation overrides namespace annotation:
    - `splunk.com/index` at pod/workload level wins
    - `splunk.com/include` / `splunk.com/exclude` can also be pod or namespace scoped

### High volume / performance issues

- If OTel agents are CPU constrained, adjust resources in `values.yaml`:

  ```yaml
  agent:
    resources:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "200m"
        memory: "256Mi"
  ```

- Monitor HEC throughput and add more Splunk indexers / tuning if backpressure is observed.

---

## 9. Cleanup

To remove the collector:

```bash
helm -n observability uninstall splunk-otel
kubectl delete namespace observability
```

(Only delete the namespace if it’s dedicated to observability and not shared.)

To remove annotations (example):

```bash
kubectl annotate namespace confluent splunk.com/exclude- splunk.com/index-
kubectl annotate deployment connect -n confluent splunk.com/include- splunk.com/index-
```

The `-` suffix removes an annotation key.

---

## 10. Summary

You now have:

- CFK‑managed **Connect** pods on Kubernetes emitting logs to stdout
- A Splunk **OpenTelemetry Collector** DaemonSet tailing container logs
- **Index‑aware routing** and include/exclude control via annotations
- Optional JSON logging for better Splunk field extraction

This setup is reusable for other CFK components (Kafka, SR, ksqlDB, etc.) by adjusting namespace and annotations accordingly.