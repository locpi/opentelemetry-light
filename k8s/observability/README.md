# Stack d'Observabilité Kubernetes (Mode Léger)

Configuration Kubernetes légère pour déployer Grafana, Tempo, Loki et Prometheus sur Minikube.

## Composants

- **Prometheus** (v2.47.0) : Collecte de métriques
- **Loki** (v2.9.0) : Agrégation de logs
- **Tempo** (v2.2.3) : Tracing distribué
- **Grafana** (v10.1.5) : Visualisation et tableaux de bord
- **OpenTelemetry Collector** (v0.91.0) : Collecte et routage de télémétrie

## Configuration Légère

Tous les composants sont configurés avec :
- **Réplication** : 1 replica par service
- **Stockage** : emptyDir (données en mémoire, non persistantes)
- **Rétention** : 2h pour Prometheus et Loki, 1h pour Tempo
- **Ressources** :
  - Requests : 100m CPU, 256Mi RAM
  - Limits : 500m CPU, 512Mi RAM

## Prérequis

- Minikube installé et démarré
- kubectl configuré

```bash
# Démarrer Minikube avec des ressources suffisantes
minikube start --cpus=4 --memory=4096
```

## Déploiement

### 1. Déployer tous les composants

```bash
kubectl apply -f k8s/opentelemetry/
```

### 2. Vérifier le déploiement

```bash
# Vérifier que tous les pods sont en cours d'exécution
kubectl get pods -n opentelemetry

# Vérifier les services
kubectl get svc -n opentelemetry
```

### 3. Accéder à Grafana

```bash
# Port-forward pour accéder à Grafana
kubectl port-forward -n opentelemetry svc/grafana 3000:3000
```

Ouvrir http://localhost:3000 dans votre navigateur.

L'authentification anonyme est activée avec le rôle Admin.

## Accès aux Services

### Grafana
```bash
kubectl port-forward -n opentelemetry svc/grafana 3000:3000
# http://localhost:3000
```

### Prometheus
```bash
kubectl port-forward -n opentelemetry svc/prometheus 9090:9090
# http://localhost:9090
```

### Tempo (OTLP endpoint pour vos applications)
```bash
# OTLP gRPC
kubectl port-forward -n opentelemetry svc/tempo 4317:4317

# OTLP HTTP
kubectl port-forward -n opentelemetry svc/tempo 4318:4318
```

### Loki
```bash
kubectl port-forward -n opentelemetry svc/loki 3100:3100
# http://localhost:3100
```

### OpenTelemetry Collector
```bash
# OTLP gRPC
kubectl port-forward -n opentelemetry svc/otel-collector 4317:4317

# OTLP HTTP
kubectl port-forward -n opentelemetry svc/otel-collector 4318:4318
```

## Configuration des Datasources Grafana

Les datasources suivantes sont préconfigurées :

1. **Prometheus** (par défaut)
   - URL: http://prometheus:9090

2. **Loki**
   - URL: http://loki:3100

3. **Tempo**
   - URL: http://tempo:3200
   - Intégrations :
     - Traces → Logs (Loki)
     - Traces → Metrics (Prometheus)
     - Service Graph
     - Node Graph

## Intégration avec vos Applications

### OpenTelemetry Collector (Recommandé)

L'OpenTelemetry Collector est déployé et prêt à recevoir les traces, métriques et logs de vos applications. Il route automatiquement :
- Les traces vers Tempo
- Les métriques vers Prometheus
- Les logs vers Loki

Il ajoute aussi automatiquement les métadonnées Kubernetes (namespace, pod, deployment, etc.).

#### Variables d'environnement OTLP

Pour vos applications instrumentées avec OpenTelemetry, pointez vers le collector :

```bash
# Via le Collector (RECOMMANDÉ - ajoute les attributs Kubernetes automatiquement)
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.opentelemetry.svc.cluster.local:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# Ou pour gRPC
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.opentelemetry.svc.cluster.local:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# Direct vers Tempo (sans enrichissement Kubernetes)
OTEL_EXPORTER_OTLP_ENDPOINT=http://tempo.opentelemetry.svc.cluster.local:4318
```

### Configuration Spring Boot avec OpenTelemetry

#### Option 1 : Java Agent (Sans modification de code)

1. Téléchargez le Java Agent OpenTelemetry dans votre image Docker :

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app

# Télécharger l'agent OpenTelemetry
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar /app/opentelemetry-javaagent.jar

# Copier votre application
COPY target/your-app.jar /app/app.jar

# Démarrer avec l'agent
ENTRYPOINT ["java", "-javaagent:/app/opentelemetry-javaagent.jar", "-jar", "/app/app.jar"]
```

2. Configurez votre deployment Kubernetes :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-spring-app
  namespace: my-namespace
spec:
  template:
    metadata:
      labels:
        app: my-spring-app
    spec:
      containers:
        - name: app
          image: my-spring-app:latest
          env:
            # Configuration OpenTelemetry
            - name: OTEL_SERVICE_NAME
              value: "my-spring-app"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.opentelemetry.svc.cluster.local:4318"
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: "http/protobuf"
            - name: OTEL_METRICS_EXPORTER
              value: "otlp"
            - name: OTEL_LOGS_EXPORTER
              value: "otlp"
            - name: OTEL_TRACES_EXPORTER
              value: "otlp"
            # Optionnel : niveau de log
            - name: OTEL_JAVAAGENT_LOGGING
              value: "application"
```

#### Option 2 : Dépendances Spring Boot (avec code)

1. Ajoutez les dépendances dans votre `pom.xml` :

```xml
<dependencies>
    <!-- OpenTelemetry Spring Boot Starter -->
    <dependency>
        <groupId>io.opentelemetry.instrumentation</groupId>
        <artifactId>opentelemetry-spring-boot-starter</artifactId>
        <version>2.0.0</version>
    </dependency>

    <!-- OTLP Exporter -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-bom</artifactId>
            <version>1.33.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>io.opentelemetry.instrumentation</groupId>
            <artifactId>opentelemetry-instrumentation-bom-alpha</artifactId>
            <version>2.0.0-alpha</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

2. Configurez dans `application.yml` :

```yaml
spring:
  application:
    name: my-spring-app

otel:
  exporter:
    otlp:
      endpoint: http://otel-collector.opentelemetry.svc.cluster.local:4318
  resource:
    attributes:
      service.name: ${spring.application.name}
      deployment.environment: minikube
  traces:
    exporter: otlp
  metrics:
    exporter: otlp
  logs:
    exporter: otlp
```

#### Option 3 : Micrometer (Métriques seulement)

Si vous utilisez déjà Micrometer dans Spring Boot :

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Et ajoutez les annotations Prometheus à votre pod (voir ci-dessous).

### Annotations Prometheus pour vos Pods

Pour que Prometheus scrape les métriques de vos applications, ajoutez ces annotations :

#### Sur le Service (recommandé pour Spring Boot)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: weeding-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"  # Pour Spring Boot Actuator
```

#### Sur le Pod/Deployment

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
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
```

### Configuration Spring Boot pour les Métriques

Pour exposer les métriques Prometheus dans Spring Boot, ajoutez dans `application.yml` :

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus,metrics,info
  metrics:
    export:
      prometheus:
        enabled: true
```

Et dans votre `pom.xml` :

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

## Jobs Prometheus Configurés

La configuration Prometheus inclut 3 jobs de scraping :

### 1. **kubernetes-pods** (Discovery automatique)
Scrape tous les pods avec les annotations `prometheus.io/scrape: "true"` dans tous les namespaces.

### 2. **backend-service** (Ciblé)
Job spécifique pour scraper `backend-service` dans le namespace `weeding-app`.
- Cible : `backend-service.weeding-app.svc.cluster.local`
- Labels ajoutés : `kubernetes_namespace`, `kubernetes_service`, `kubernetes_pod_name`

### 3. **kubernetes-services** (Services annotés)
Scrape tous les services Kubernetes avec les annotations `prometheus.io/scrape: "true"`.

### Vérifier que Prometheus scrape votre service

```bash
# Port-forward vers Prometheus
kubectl port-forward -n opentelemetry svc/prometheus 9090:9090

# Ouvrir http://localhost:9090
# Aller dans Status > Targets
# Chercher le job "backend-service" ou "kubernetes-pods"
```

### Exemple de requêtes PromQL pour backend-service

```promql
# CPU usage
rate(process_cpu_usage{kubernetes_service="backend-service"}[5m])

# Memory usage
jvm_memory_used_bytes{kubernetes_service="backend-service"}

# HTTP requests
rate(http_server_requests_seconds_count{kubernetes_service="backend-service"}[5m])

# HTTP latency p95
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{kubernetes_service="backend-service"}[5m]))
```

## Nettoyage

```bash
# Supprimer tous les composants
kubectl delete -f k8s/opentelemetry/

# Ou supprimer le namespace complet
kubectl delete namespace opentelemetry
```

## Limitations (Mode Léger)

- Stockage non persistant (données perdues au redémarrage des pods)
- Rétention courte (2h max)
- Pas de haute disponibilité
- Ressources CPU et mémoire limitées
- Non recommandé pour la production

## Passer en Mode Production

Pour un environnement de production :

1. Utiliser des PersistentVolumeClaims au lieu d'emptyDir
2. Augmenter les replicas
3. Augmenter la rétention des données
4. Ajouter des ressources CPU/mémoire
5. Configurer l'authentification Grafana
6. Ajouter des alertes Prometheus
7. Mettre en place des backups

## Exemple Complet : Déploiement Spring Boot

Voici un exemple complet de déploiement d'une application Spring Boot avec OpenTelemetry :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-demo
  namespace: default
  labels:
    app: spring-boot-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-boot-demo
  template:
    metadata:
      labels:
        app: spring-boot-demo
        version: v1.0.0
      annotations:
        # Pour Prometheus scraping (si vous exposez /actuator/prometheus)
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
        - name: app
          image: your-spring-app:latest
          ports:
            - containerPort: 8080
              name: http
          env:
            # Configuration Spring Boot
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"

            # Configuration OpenTelemetry
            - name: OTEL_SERVICE_NAME
              value: "spring-boot-demo"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.opentelemetry.svc.cluster.local:4318"
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: "http/protobuf"
            - name: OTEL_METRICS_EXPORTER
              value: "otlp"
            - name: OTEL_LOGS_EXPORTER
              value: "otlp"
            - name: OTEL_TRACES_EXPORTER
              value: "otlp"

            # Attributs de ressource personnalisés
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "deployment.environment=minikube,service.version=v1.0.0"

          resources:
            requests:
              cpu: 200m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi

---
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-demo
  namespace: default
spec:
  selector:
    app: spring-boot-demo
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

## Vérification de l'Intégration

Une fois votre application déployée :

1. **Vérifier les logs du collector** :
```bash
kubectl logs -n opentelemetry deployment/otel-collector -f
```

2. **Vérifier les traces dans Tempo** :
   - Accédez à Grafana (http://localhost:3000)
   - Allez dans Explore
   - Sélectionnez la datasource "Tempo"
   - Recherchez vos traces par service name

3. **Vérifier les métriques dans Prometheus** :
```bash
kubectl port-forward -n opentelemetry svc/prometheus 9090:9090
```
   - Ouvrez http://localhost:9090
   - Recherchez vos métriques : `otel_*` ou votre service name

4. **Vérifier les logs dans Loki** :
   - Dans Grafana, allez dans Explore
   - Sélectionnez "Loki"
   - Utilisez des requêtes comme : `{k8s_namespace_name="default", k8s_pod_name=~"spring-boot-demo.*"}`

## Troubleshooting

### L'application ne peut pas joindre le collector

Vérifiez que le service est accessible depuis votre namespace :

```bash
# Depuis un pod de votre namespace
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -v http://otel-collector.opentelemetry.svc.cluster.local:4318/v1/traces
```

### Pas de traces visibles dans Tempo

1. Vérifiez les logs du collector
2. Vérifiez que `OTEL_TRACES_EXPORTER=otlp` est bien défini
3. Vérifiez que votre application génère du trafic

### Pas de métriques dans Prometheus

1. Vérifiez les annotations Prometheus sur vos pods
2. Vérifiez les logs de Prometheus pour voir s'il scrape votre application
3. Pour l'export OTLP, vérifiez les logs du collector

## Ressources Utiles

- [Documentation Prometheus](https://prometheus.io/docs/)
- [Documentation Loki](https://grafana.com/docs/loki/latest/)
- [Documentation Tempo](https://grafana.com/docs/tempo/latest/)
- [Documentation Grafana](https://grafana.com/docs/grafana/latest/)
- [OpenTelemetry](https://opentelemetry.io/)
- [OpenTelemetry Java Instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
