# OpenTelemetry Light - Stack d'Observabilité pour Kubernetes

Stack complète d'observabilité légère pour Minikube avec auto-instrumentation Java.

## 🎯 Fonctionnalités

- ✅ **Prometheus** : Métriques (CPU, RAM, JVM, HTTP, etc.)
- ✅ **Loki** : Agrégation et recherche de logs
- ✅ **Tempo** : Tracing distribué avec corrélation
- ✅ **Grafana** : Visualisation unifiée (traces, métriques, logs)
- ✅ **OpenTelemetry Collector** : Collecte et routage avec enrichissement Kubernetes
- ✅ **Auto-instrumentation Java** : Capture automatique de tous les appels (REST, BDD, async, etc.)

## 🚀 Démarrage Rapide

```bash
# 1. Démarrer Minikube
minikube start --cpus=4 --memory=4096

# 2. Déployer la stack d'observabilité
kubectl apply -f k8s/observability/

# 3. Installer l'auto-instrumentation Java
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
kubectl apply -f k8s/instrumentation/java-auto-instrumentation.yaml

# 4. Activer l'instrumentation sur vos applications
# Ajoutez cette annotation à vos Deployments :
# instrumentation.opentelemetry.io/inject-java: "true"

# 5. Accéder à Grafana
kubectl port-forward -n opentelemetry svc/grafana 3000:3000
# Ouvrir http://localhost:3000
```

**Voir le guide complet** : [QUICK_START.md](./QUICK_START.md)

## 📊 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Votre Application Java                    │
│  (backend-service, auth-service, profile-service, etc.)      │
│                                                               │
│  📝 Auto-instrumenté avec OpenTelemetry Java Agent          │
│     • Traces : Appels REST, BDD, async                       │
│     • Métriques : JVM, HTTP, custom                          │
│     • Logs : Application logs                                │
└──────────────────────┬──────────────────────────────────────┘
                       │ OTLP (gRPC/HTTP)
                       ↓
┌─────────────────────────────────────────────────────────────┐
│          OpenTelemetry Collector (namespace: opentelemetry)  │
│                                                               │
│  🔄 Traitement :                                             │
│     • Enrichissement avec métadonnées Kubernetes             │
│     • Batching et filtrage                                   │
│     • Routage vers les backends                              │
└──────────────┬────────────────┬────────────────┬────────────┘
               │                 │                 │
       Traces  │         Metrics │         Logs    │
               ↓                 ↓                 ↓
         ┌─────────┐      ┌──────────┐      ┌─────────┐
         │  Tempo  │      │Prometheus│      │  Loki   │
         └────┬────┘      └─────┬────┘      └────┬────┘
              │                  │                 │
              └──────────────────┴─────────────────┘
                                 │
                                 ↓
                          ┌──────────────┐
                          │   Grafana    │
                          │              │
                          │ Visualisation│
                          │   Unifiée    │
                          └──────────────┘
```

## 📁 Structure du Projet

```
.
├── k8s/
│   ├── observability/           # Stack d'observabilité
│   │   ├── 00-namespace.yaml    # Namespace opentelemetry
│   │   ├── 01-prometheus.yaml   # Prometheus + scraping config
│   │   ├── 02-loki.yaml         # Loki pour les logs
│   │   ├── 03-tempo.yaml        # Tempo pour les traces
│   │   ├── 04-grafana.yaml      # Grafana + datasources
│   │   ├── 05-otel-collector.yaml # OpenTelemetry Collector
│   │   └── README.md            # Documentation détaillée
│   │
│   ├── instrumentation/         # Auto-instrumentation Java
│   │   ├── java-auto-instrumentation.yaml
│   │   └── README.md            # Guide d'instrumentation
│   │
│   └── examples/                # Exemples de configuration
│       ├── backend-service-example.yaml
│       └── backend-service-instrumented.yaml
│
├── QUICK_START.md               # Guide de démarrage rapide
└── README.md                    # Ce fichier
```

## 🔧 Configuration

### Pour scraper les métriques Prometheus de vos services

Ajoutez ces annotations à votre Service ou Deployment :

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"
```

### Pour capturer les traces de vos applications Java

**Option 1 : Auto-instrumentation (Recommandé)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-service
  namespace: weeding-app
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
    spec:
      containers:
        - name: app
          env:
            - name: OTEL_SERVICE_NAME
              value: "backend-service"
```

**Option 2 : Manuel (via Dockerfile)**

```dockerfile
FROM eclipse-temurin:17-jre
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar /app/opentelemetry-javaagent.jar
COPY target/app.jar /app/app.jar
ENV JAVA_TOOL_OPTIONS="-javaagent:/app/opentelemetry-javaagent.jar"
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

## 📈 Ce qui est automatiquement capturé

### Traces
- ✅ Endpoints REST (Spring MVC, Spring WebFlux)
- ✅ Appels HTTP entre services (RestTemplate, WebClient, OkHttp)
- ✅ Requêtes BDD (JDBC, JPA, Hibernate, MongoDB, Redis)
- ✅ Messaging (Kafka, RabbitMQ, JMS)
- ✅ Code asynchrone (CompletableFuture, Reactor, RxJava)

### Métriques
- ✅ Métriques JVM (heap, threads, GC)
- ✅ Métriques HTTP (requests, latency, errors)
- ✅ Métriques custom Spring Boot Actuator
- ✅ Métriques système (CPU, RAM)

### Logs
- ✅ Logs applicatifs
- ✅ Corrélation avec les traces (trace_id, span_id)
- ✅ Enrichissement avec métadonnées Kubernetes

## 🎓 Guides et Documentation

- **[Guide de Démarrage Rapide](./QUICK_START.md)** - Installation et premiers pas
- **[Configuration Observabilité](./k8s/observability/README.md)** - Détails sur la stack
- **[Auto-Instrumentation Java](./k8s/instrumentation/README.md)** - Guide complet d'instrumentation
- **[Exemples](./k8s/examples/)** - Configurations d'exemple

## 🐛 Troubleshooting

### Je ne vois que les traces BDD, pas les appels entre services

➡️ **Solution** : Activez l'auto-instrumentation complète avec l'annotation `instrumentation.opentelemetry.io/inject-java: "true"` sur vos Deployments.

Voir : [k8s/instrumentation/README.md](./k8s/instrumentation/README.md)

### Pas de traces visibles dans Grafana

1. Vérifier que l'application génère du trafic
2. Vérifier l'injection du Java Agent : `kubectl describe pod <pod-name> | grep JAVA_TOOL_OPTIONS`
3. Vérifier les logs du collector : `kubectl logs -n opentelemetry deployment/otel-collector`

### Le pod ne démarre pas après l'injection

➡️ **Cause** : Mémoire insuffisante. Le Java Agent ajoute ~100MB.

➡️ **Solution** : Augmenter les ressources :
```yaml
resources:
  requests:
    memory: 768Mi
  limits:
    memory: 1536Mi
```

### Prometheus ne scrape pas mon service

1. Vérifier les annotations : `prometheus.io/scrape: "true"`
2. Vérifier que l'endpoint expose des métriques : `curl http://service:port/actuator/prometheus`
3. Vérifier les targets dans Prometheus : http://localhost:9090/targets

## 🔗 Accès aux Services

| Service | Port Forward | URL |
|---------|-------------|-----|
| Grafana | `kubectl port-forward -n opentelemetry svc/grafana 3000:3000` | http://localhost:3000 |
| Prometheus | `kubectl port-forward -n opentelemetry svc/prometheus 9090:9090` | http://localhost:9090 |
| OTLP Collector | `kubectl port-forward -n opentelemetry svc/otel-collector 4318:4318` | http://localhost:4318 |

## 🎯 Exemple de Requêtes

### PromQL (Prometheus)

```promql
# Taux de requêtes HTTP
rate(http_server_requests_seconds_count{kubernetes_service="backend-service"}[5m])

# Latence p95
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{kubernetes_service="backend-service"}[5m]))

# Mémoire JVM
jvm_memory_used_bytes{kubernetes_service="backend-service",area="heap"}
```

### LogQL (Loki)

```logql
# Logs d'un service
{k8s_namespace_name="weeding-app", k8s_pod_name=~"backend-service.*"}

# Logs avec erreurs
{k8s_namespace_name="weeding-app"} |= "ERROR"

# Logs d'une trace spécifique
{k8s_namespace_name="weeding-app"} | json | trace_id="abc123"
```

## ⚙️ Configuration Avancée

### Ajouter des spans personnalisés

```java
import io.opentelemetry.instrumentation.annotations.WithSpan;
import io.opentelemetry.instrumentation.annotations.SpanAttribute;

@Service
public class UserService {

    @WithSpan("getUserById")
    public User getUser(@SpanAttribute("user.id") Long userId) {
        // Cette méthode sera automatiquement tracée
        return userRepository.findById(userId);
    }
}
```

### Filtrer les traces (sampling)

Pour réduire le volume en production, modifiez `java-auto-instrumentation.yaml` :

```yaml
env:
  - name: OTEL_TRACES_SAMPLER
    value: "traceidratio"
  - name: OTEL_TRACES_SAMPLER_ARG
    value: "0.1"  # Capturer 10% des traces
```

## 📊 Dashboards Grafana Recommandés

- **JVM Metrics** : ID `4701` (via grafana.com)
- **Spring Boot** : ID `12900`
- **Kubernetes Cluster Monitoring** : ID `315`

Importer via : Grafana → Dashboards → Import → ID

## 🤝 Contribution

Les contributions sont bienvenues ! N'hésitez pas à ouvrir une issue ou une PR.

## 📝 License

MIT

## 🔗 Ressources

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [OpenTelemetry Java Instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation)
- [Supported Java Libraries](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md)
