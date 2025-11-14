# Guide de Démarrage Rapide - Observabilité complète

Ce guide vous permet de déployer rapidement une stack d'observabilité complète et d'instrumenter vos applications Java pour capturer **tous les appels entre services**.

## 🚀 Installation Rapide

### 1. Démarrer Minikube

```bash
minikube start --cpus=4 --memory=4096
```

### 2. Installer la Stack d'Observabilité

```bash
# Créer le namespace
kubectl apply -f k8s/observability/00-namespace.yaml

# Déployer Prometheus, Loki, Tempo, Grafana et OpenTelemetry Collector
kubectl apply -f k8s/observability/

# Attendre que tous les pods soient prêts
kubectl wait --for=condition=ready pod -l app=grafana -n opentelemetry --timeout=300s
kubectl wait --for=condition=ready pod -l app=prometheus -n opentelemetry --timeout=300s
kubectl wait --for=condition=ready pod -l app=loki -n opentelemetry --timeout=300s
kubectl wait --for=condition=ready pod -l app=tempo -n opentelemetry --timeout=300s
kubectl wait --for=condition=ready pod -l app=otel-collector -n opentelemetry --timeout=300s
```

### 3. Installer l'Auto-Instrumentation Java

```bash
# Installer cert-manager (prérequis)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Attendre que cert-manager soit prêt
kubectl wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=300s

# Installer l'OpenTelemetry Operator
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

# Attendre que l'operator soit prêt
kubectl wait --for=condition=available deployment/opentelemetry-operator-controller-manager \
  -n opentelemetry-operator-system --timeout=300s

# Déployer les configurations d'instrumentation
kubectl apply -f k8s/instrumentation/java-auto-instrumentation.yaml
```

### 4. Instrumenter vos Applications

#### Option A : Avec l'OpenTelemetry Operator (Recommandé)

Ajoutez simplement cette annotation à vos Deployments :

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
        # ⭐ Cette ligne active l'instrumentation automatique
        instrumentation.opentelemetry.io/inject-java: "true"
    spec:
      containers:
        - name: app
          image: your-app:latest
          env:
            - name: OTEL_SERVICE_NAME
              value: "backend-service"
```

Puis redémarrez :

```bash
kubectl rollout restart deployment/backend-service -n weeding-app
```

#### Option B : Sans Operator (Manuel)

Modifiez votre Dockerfile :

```dockerfile
FROM eclipse-temurin:17-jre

# Télécharger le Java Agent
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar \
    /app/opentelemetry-javaagent.jar

COPY target/app.jar /app/app.jar

ENV JAVA_TOOL_OPTIONS="-javaagent:/app/opentelemetry-javaagent.jar"
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Et configurez votre Deployment :

```yaml
env:
  - name: OTEL_SERVICE_NAME
    value: "backend-service"
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector.opentelemetry.svc.cluster.local:4318"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "http/protobuf"
```

### 5. Accéder à Grafana

```bash
# Port-forward vers Grafana
kubectl port-forward -n opentelemetry svc/grafana 3000:3000
```

Ouvrez http://localhost:3000 dans votre navigateur.

## 📊 Vérifier que ça fonctionne

### Voir les Traces

1. Dans Grafana, cliquez sur **Explore** (icône boussole)
2. Sélectionnez la datasource **Tempo**
3. Dans "Query type", sélectionnez **Search**
4. Cherchez par "Service Name" : `backend-service`
5. Cliquez sur une trace pour voir le détail

Vous devriez voir :
- ✅ **HTTP server spans** : vos endpoints REST
- ✅ **HTTP client spans** : appels vers d'autres services
- ✅ **JDBC spans** : requêtes SQL
- ✅ **Method spans** : vos méthodes métier

### Voir les Métriques

1. Dans Grafana **Explore**
2. Sélectionnez **Prometheus**
3. Exemples de requêtes :

```promql
# Taux de requêtes HTTP
rate(http_server_requests_seconds_count{kubernetes_service="backend-service"}[5m])

# Latence p95
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{kubernetes_service="backend-service"}[5m]))

# Utilisation CPU
rate(process_cpu_usage{kubernetes_service="backend-service"}[5m])

# Mémoire JVM
jvm_memory_used_bytes{kubernetes_service="backend-service"}
```

### Voir les Logs

1. Dans Grafana **Explore**
2. Sélectionnez **Loki**
3. Requête exemple :

```logql
{k8s_namespace_name="weeding-app", k8s_pod_name=~"backend-service.*"}
```

## 🔍 Exemple de Trace Complète

Avec l'auto-instrumentation activée, voici ce que vous verrez pour un endpoint typique :

```
backend-service: GET /api/users/123 (250ms)
  ├─ HTTP GET auth-service:8081/validate (50ms)
  │   └─ auth-service: POST /validate (45ms)
  │       └─ JDBC: SELECT * FROM tokens (10ms)
  ├─ UserService.getUser() (180ms)
  │   ├─ UserRepository.findById() (30ms)
  │   │   └─ JDBC: SELECT * FROM users WHERE id=123 (25ms)
  │   └─ HTTP GET profile-service:8082/profiles/123 (140ms)
  │       └─ profile-service: GET /profiles/123 (135ms)
  │           └─ JDBC: SELECT * FROM profiles (30ms)
  └─ Response 200 OK
```

## 📝 Ce qui est automatiquement tracé

### ✅ Frameworks Web
- Spring MVC / Spring Boot
- Spring WebFlux (reactive)
- Java Servlets

### ✅ Clients HTTP (appels entre services)
- RestTemplate
- WebClient
- OkHttp
- Apache HttpClient
- Java 11 HttpClient

### ✅ Bases de Données
- JDBC
- JPA / Hibernate
- MongoDB
- Redis

### ✅ Messaging
- Kafka
- RabbitMQ
- JMS

### ✅ Code Asynchrone
- ExecutorService / CompletableFuture
- Project Reactor
- RxJava

## 🛠️ Troubleshooting

### Pas de traces dans Grafana ?

1. **Vérifier que l'application génère du trafic**
   ```bash
   # Faire quelques requêtes vers votre service
   kubectl run -it --rm curl --image=curlimages/curl --restart=Never -- \
     curl http://backend-service.weeding-app:8080/api/health
   ```

2. **Vérifier l'injection du Java Agent**
   ```bash
   kubectl describe pod -n weeding-app -l app=backend-service | grep -i java
   # Vous devriez voir : JAVA_TOOL_OPTIONS=-javaagent:/otel-auto-instrumentation/javaagent.jar
   ```

3. **Vérifier les logs du collector**
   ```bash
   kubectl logs -n opentelemetry deployment/otel-collector -f
   ```

4. **Vérifier les logs de l'application**
   ```bash
   kubectl logs -n weeding-app -l app=backend-service | grep -i otel
   # Au démarrage vous devriez voir : [otel.javaagent] OpenTelemetry Javaagent enabled
   ```

### Pas d'appels entre services dans les traces ?

1. **Vérifier que TOUS les services sont instrumentés**
   - Chaque service doit avoir l'annotation `instrumentation.opentelemetry.io/inject-java: "true"`

2. **Vérifier la configuration des propagateurs**
   ```bash
   kubectl get instrumentation -n weeding-app -o yaml | grep -A 5 propagators
   # Doit inclure : tracecontext, baggage
   ```

3. **Vérifier que les services utilisent HTTP**
   - Les appels directs de méthodes (même JVM) ne sont pas tracés comme des appels entre services
   - Utilisez RestTemplate, WebClient, ou Feign

### Le pod ne démarre pas ?

**Problème fréquent : Mémoire insuffisante**

Le Java Agent ajoute ~100MB de mémoire. Augmentez les ressources :

```yaml
resources:
  requests:
    memory: 768Mi  # Au lieu de 512Mi
  limits:
    memory: 1536Mi  # Au lieu de 1Gi
```

## 📚 Documentation Complète

- **Configuration d'observabilité** : `k8s/observability/README.md`
- **Auto-instrumentation Java** : `k8s/instrumentation/README.md`
- **Exemples** : `k8s/examples/`

## 🎯 Prochaines Étapes

1. **Créer des dashboards Grafana** pour visualiser vos métriques
2. **Configurer des alertes Prometheus** sur les erreurs et latences
3. **Ajouter des spans personnalisés** avec `@WithSpan` pour vos méthodes critiques
4. **Optimiser le sampling** en production pour réduire le volume

## 💡 Astuces

### Voir tous les services tracés

Dans Grafana Explore (Tempo), utilisez la requête :
```
{}
```

Puis regardez les suggestions dans "Service Name".

### Rechercher des traces avec erreurs

Dans Tempo :
- Query type : **Search**
- Status : **Error**

### Corrélation Traces → Logs

Dans une trace Tempo, cliquez sur un span puis sur "Logs for this span" pour voir les logs correspondants dans Loki.

### Service Graph

Dans Grafana, créez un dashboard avec le panel "Node Graph" et la datasource Tempo pour visualiser la topologie de vos services.
