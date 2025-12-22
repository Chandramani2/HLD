# Implementing Logging Sidecar for Spring Boot

This document outlines how to decouple logging from a Spring Boot application using the **Sidecar Pattern** in a Kubernetes environment.

---

## 📋 Architecture Overview

The Spring Boot application writes logs to a local file system. A secondary container (Sidecar) runs in the same Pod, reads that file, and streams it to a centralized logging platform.



---

## 1️⃣ Spring Boot Configuration
Configure the application to output logs to a file rather than just the console.

**`src/main/resources/application.yml`**
```yaml
logging:
  file:
    name: /var/log/app/spring-boot.log
  logback:
    rollingpolicy:
      max-file-size: 10MB
      total-size-cap: 100MB
      max-history: 7
```

## 2️⃣ Sidecar Configuration (Filebeat)

The sidecar acts as the "shipper." It is configured to watch the directory where the Spring Boot app writes its logs. By using a sidecar, you don't have to configure network protocols or SSL certificates inside your Java code.

**`filebeat.yml`**
```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /usr/share/filebeat/logs/*.log # Matches the volume mount path
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}' # Handles Java Stack Traces
  multiline.negate: true
  multiline.match: after

output.elasticsearch:
  hosts: ["elasticsearch-logging:9200"]
  protocol: "https"
```

## 3️⃣ Kubernetes Deployment

To implement the sidecar pattern, both the primary application and the auxiliary service must be defined within the same **Pod spec**. They share an `emptyDir` volume, which is a temporary directory that exists for the lifetime of the Pod and allows the two containers to communicate via the file system.



**`deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-sidecar-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        # 🟢 Container 1: The Main Spring Boot App
        - name: spring-app
          image: my-docker-repo/spring-app:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: shared-log-volume
              mountPath: /var/log/app # Must match 'logging.file.name' in application.yml

        # 🔵 Container 2: The Filebeat Sidecar
        - name: filebeat-sidecar
          image: docker.elastic.co/beats/filebeat:8.x.x
          volumeMounts:
            - name: shared-log-volume
              mountPath: /usr/share/filebeat/logs
              readOnly: true
            - name: filebeat-config
              mountPath: /usr/share/filebeat/filebeat.yml
              subPath: filebeat.yml

      # 📂 Shared Volume definition
      volumes:
        - name: shared-log-volume
          emptyDir: {} 
        - name: filebeat-config
          configMap:
            name: filebeat-config-map
```
## 🧪 Testing the Implementation

Once you have deployed your manifest, you can verify that the Sidecar pattern is functioning correctly by interacting with the Kubernetes cluster.

### 1. Verify Pod Status
Check if both containers (the main app and the sidecar) are running within the same pod. You should see `2/2` in the **READY** column.

```bash
kubectl get pods -l app=spring-app
```
### 2. Inspect Sidecar Logs
To ensure the sidecar is successfully "harvesting" the logs created by Spring Boot, stream the logs specifically from the `filebeat-sidecar` container. This confirms the sidecar can see the files in the shared volume.

```bash
# Target the sidecar container specifically using the -c flag
kubectl logs <your-pod-name> -c filebeat-sidecar
```
### 3. Generate Traffic
Trigger activity in your Spring Boot application by hitting its REST endpoints. The sidecar should immediately detect the new log entries in the shared volume and ship them to your configured destination (such as Elasticsearch, Logstash, or Kafka).



---

## 🚀 Why This Matters for System Design

The Sidecar pattern is a fundamental building block in modern cloud-native architecture for several key reasons:

* **Isolation of Failures:** If the logging agent (Filebeat) crashes or hangs due to a network partition, the primary Spring Boot application remains completely unaffected and continues to serve user requests.
* **Reduced Application Footprint:** You don't need to bundle heavy logging SDKs or complex Java dependencies (like specialized Logback appenders) into your main application JAR file.
* **Security & Compliance:** Sensitive credentials (like API keys or SSL certificates for the logging server) are mounted only to the sidecar, keeping them hidden from the main application's environment.
* **Standardization:** Your infrastructure team can push the same logging sidecar to Go, Python, or Node.js services, ensuring a unified logging format across the entire organization without code changes.

---

> **Final Pro-Tip:** When using the sidecar pattern for logging, always ensure you configure **Log Rotation** in your Spring Boot application (using `rollingpolicy`). Since the `emptyDir` volume uses the node's storage, an unmanaged log file could eventually fill up the disk space on your Kubernetes Worker Node.



