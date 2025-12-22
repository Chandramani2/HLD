# Service Discovery Implementation Examples

Service discovery can be implemented either at the **Application Level** (using libraries) or at the **Platform Level** (using infrastructure like Kubernetes).

---

## 🍃 1. Netflix Eureka (Client-Side Discovery)

In the Spring Boot ecosystem, service discovery is "code-aware." The application itself handles registration and fetching the registry.

### A. The Eureka Server (The Phonebook)
This acts as the central hub where all services check in.

```java
@SpringBootApplication
@EnableEurekaServer // Turns this app into a Service Registry
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

### B. The Eureka Client (The Service)
Every microservice (e.g., "Payment Service") registers its own IP and Port.

**`application.properties`:**
```properties
spring.application.name=payment-service
server.port=8081
# Address of the central phonebook
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka
```

`PaymentApplication.java:`
```java
@SpringBootApplication
@EnableDiscoveryClient // Automatically heartbeats to Eureka on startup
public class PaymentApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentApplication.class, args);
    }
}
```

## ☸️ 2. Kubernetes (Server-Side Discovery)

In Kubernetes, discovery is **transparent**. The application doesn't need to know a registry exists; it just uses standard DNS.

### The Service Definition (`service.yaml`)
This creates a permanent DNS entry (`auth-service`) that points to a group of dynamic Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service  # The "domain name" other apps will call
spec:
  selector:
    app: auth-app     # Automatically finds Pods with this label
  ports:
    - protocol: TCP
      port: 80        # The stable port for the service
      targetPort: 9000 # The actual port the app runs on
```
### How the Client Calls It
The developer doesn't need any special libraries. They just call the service name as if it were a local website or internal domain.

```javascript
// Kubernetes DNS maps "auth-service" to a healthy Pod automatically
fetch('http://auth-service/login')
  .then(response => console.log('Response received!'));
```
## 📊 Summary Comparison

| Feature | **Netflix Eureka** | **Kubernetes** |
| :--- | :--- | :--- |
| **Discovery Logic** | Inside the application code. | Handled by the Infrastructure. |
| **Language Support** | Best for Java/Spring Boot. | Language Agnostic (Any app works). |
| **Setup** | Must manage a Eureka Server app. | Built-in to the cluster. |
| **Load Balancing** | Client-side (Ribbon/LoadBalancer). | Server-side (Kube-Proxy). |
| **Visibility** | Devs see the registration logic. | Completely invisible to the dev. |

[Image of service discovery patterns client-side vs server-side]

---

### Final Takeaway for System Design
When designing a system, choosing between these implementation styles depends on your environment:
* **Use Application-Level (Eureka):** If you need fine-grained control over load-balancing algorithms or are running in a non-containerized legacy environment.
* **Use Platform-Level (Kubernetes):** If you want a "hands-off" approach where the infrastructure handles scaling, health checks, and routing for any programming language automatically.

---