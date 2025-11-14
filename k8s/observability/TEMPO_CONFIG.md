# Configuration Tempo pour les Spans Java

## ✅ Améliorations Apportées

Votre configuration Tempo a été optimisée pour mieux afficher les spans des applications Java.

### 1. **Configuration des Receivers OTLP**

```yaml
distributor:
  receivers:
    otlp:
      protocols:
        http:
          endpoint: 0.0.0.0:4318  # Explicitement défini
        grpc:
          endpoint: 0.0.0.0:4317  # Explicitement défini
```

✅ Les endpoints OTLP sont maintenant explicites pour recevoir les traces Java.

### 2. **Optimisation de l'Ingester pour Java**

```yaml
ingester:
  max_block_duration: 5m
  trace_idle_period: 20s       # ⭐ Nouveau : Temps d'attente avant de compléter une trace
  max_block_bytes: 1_000_000   # ⭐ Nouveau : Taille max des blocs
```

**Pourquoi ?** Les traces Java peuvent avoir beaucoup de spans (REST + BDD + méthodes). Ces paramètres permettent de :
- Attendre plus longtemps que tous les spans arrivent avant de finaliser la trace
- Gérer des traces plus volumineuses

### 3. **Metrics Generator Amélioré**

```yaml
metrics_generator:
  processor:
    service_graphs:
      dimensions:
        - http.method          # GET, POST, etc.
        - http.status_code     # 200, 404, 500, etc.
        - service.namespace    # weeding-app, etc.

    span_metrics:
      dimensions:
        - http.method
        - http.status_code
        - service.name
        - service.namespace
        - db.system           # ⭐ PostgreSQL, MySQL, MongoDB
        - db.name             # ⭐ Nom de la BDD
      enable_target_info: true
```

**Résultat** : Vous pouvez maintenant :
- Visualiser le graphe des services (qui appelle qui)
- Voir les métriques par méthode HTTP et statut
- Filtrer par base de données utilisée

### 4. **Limites Augmentées pour Java**

```yaml
overrides:
  defaults:
    ingestion:
      max_bytes_per_trace: 5_000_000      # ⭐ 5 MB (au lieu de ~1 MB par défaut)
      max_spans_per_trace: 100_000        # ⭐ 100k spans max

    metrics_generator:
      max_trace_duration: 5m              # ⭐ Traces jusqu'à 5 minutes
```

**Pourquoi ?** Les applications Java instrumentées peuvent générer beaucoup de spans :
- Un endpoint REST qui appelle 3 services et fait 5 requêtes BDD = ~50 spans minimum
- Avec l'auto-instrumentation, chaque méthode peut créer un span

### 5. **Configuration Query pour Grafana**

```yaml
query_frontend:
  search:
    max_duration: 0s              # Pas de limite de durée de recherche
    default_result_limit: 20      # 20 traces par défaut
  trace_by_id:
    query_timeout: 30s            # Timeout de 30s pour chercher une trace

querier:
  max_concurrent_queries: 10      # 10 requêtes simultanées max
```

**Résultat** : Recherche fluide dans Grafana même avec beaucoup de traces.

## 🎯 Configuration Grafana Améliorée

### Tags Kubernetes Automatiques

```yaml
tracesToLogs:
  tags: ['service.name', 'k8s.namespace.name', 'k8s.pod.name', 'k8s.deployment.name']
  mappedTags:
    - key: 'service.name'
      value: 'service'
    - key: 'k8s.namespace.name'
      value: 'namespace'
  mapTagNamesEnabled: true
```

**Résultat** : Depuis une trace, vous pouvez cliquer pour voir les logs du pod exact.

### Métriques depuis les Spans

```yaml
tracesToMetrics:
  queries:
    - name: 'Request Rate'
      query: 'rate(tempo_spanmetrics_calls_total{$__tags}[5m])'
    - name: 'Request Duration p95'
      query: 'histogram_quantile(0.95, rate(tempo_spanmetrics_latency_bucket{$__tags}[5m]))'
    - name: 'Error Rate'
      query: 'rate(tempo_spanmetrics_calls_total{$__tags, status_code="STATUS_CODE_ERROR"}[5m])'
```

**Résultat** : Depuis une trace dans Grafana, vous voyez directement les métriques RED (Rate, Errors, Duration).

### Corrélation Logs ↔ Traces

```yaml
# Dans Loki datasource
derivedFields:
  - datasourceUid: tempo
    matcherRegex: "trace_id=(\\w+)"
    name: TraceID
    url: "$${__value.raw}"
```

**Résultat** : Dans les logs, un lien cliquable vers la trace correspondante.

## 📊 Ce Que Vous Verrez Maintenant dans Grafana

### 1. Trace Complète d'un Endpoint Java

```
backend-service: GET /api/users/123 (245ms)
  │
  ├─ Spring MVC: GET /api/users/{id} (240ms)
  │   │
  │   ├─ HTTP Client: GET auth-service/validate (45ms)
  │   │   │
  │   │   └─ auth-service: POST /validate (40ms)
  │   │       │
  │   │       └─ JDBC: SELECT * FROM tokens WHERE... (8ms)
  │   │
  │   ├─ UserService.getUser() (180ms)
  │   │   │
  │   │   ├─ JPA: UserRepository.findById() (35ms)
  │   │   │   │
  │   │   │   └─ JDBC: SELECT * FROM users WHERE id=? (30ms)
  │   │   │
  │   │   └─ HTTP Client: GET profile-service/profiles/123 (140ms)
  │   │       │
  │   │       └─ profile-service: GET /profiles/{id} (135ms)
  │   │           │
  │   │           └─ JDBC: SELECT * FROM profiles WHERE... (32ms)
  │   │
  │   └─ JSON Serialization (5ms)
  │
  └─ Response: 200 OK
```

### 2. Tags Disponibles pour Recherche

Vous pouvez rechercher les traces par :
- **service.name** : `backend-service`, `auth-service`
- **http.method** : `GET`, `POST`, `PUT`, `DELETE`
- **http.status_code** : `200`, `404`, `500`
- **http.url** : `/api/users/123`
- **db.system** : `postgresql`, `mysql`, `mongodb`
- **db.name** : Nom de votre base
- **k8s.namespace.name** : `weeding-app`
- **k8s.pod.name** : `backend-service-abc123`
- **error** : `true` (pour les spans en erreur)

### 3. Service Graph

Dans Grafana, vous pouvez visualiser :
```
┌──────────────┐
│ frontend-app │
└──────┬───────┘
       │
       ↓
┌──────────────────┐         ┌──────────────┐
│ backend-service  │────────→│ auth-service │
└────────┬─────────┘         └──────────────┘
         │
         ↓
┌──────────────────┐
│ profile-service  │
└──────────────────┘
```

Avec métriques sur chaque lien :
- Nombre de requêtes/sec
- Latence moyenne
- Taux d'erreur

## 🔍 Comment Utiliser dans Grafana

### Rechercher des Traces

1. **Explore** → Datasource **Tempo**
2. **Query type** : Search
3. Filtrer par :
   - **Service Name** : `backend-service`
   - **Span Name** : `GET /api/users/{id}`
   - **Duration** : `> 200ms` (traces lentes)
   - **Status** : `error` (traces avec erreurs)

### Voir le Service Graph

1. **Explore** → Datasource **Tempo**
2. **Query type** : Service Graph
3. Sélectionner la période
4. Voir le graphe des relations entre services

### Corréler Trace → Logs

1. Ouvrir une trace dans Tempo
2. Cliquer sur un span
3. Dans le panneau de droite : **Logs for this span**
4. Voir les logs du pod exact au moment du span

### Corréler Trace → Métriques

1. Ouvrir une trace dans Tempo
2. Cliquer sur **Related Metrics**
3. Voir les requêtes PromQL avec les valeurs du span

## 🚀 Déployer les Changements

```bash
# Appliquer la nouvelle configuration
kubectl apply -f k8s/observability/03-tempo.yaml
kubectl apply -f k8s/observability/04-grafana.yaml

# Redémarrer Tempo pour prendre en compte la config
kubectl rollout restart deployment/tempo -n opentelemetry

# Redémarrer Grafana pour les datasources
kubectl rollout restart deployment/grafana -n opentelemetry

# Attendre que les pods soient prêts
kubectl wait --for=condition=ready pod -l app=tempo -n opentelemetry --timeout=300s
kubectl wait --for=condition=ready pod -l app=grafana -n opentelemetry --timeout=300s
```

## ✅ Vérification

### 1. Vérifier que Tempo reçoit les traces

```bash
# Voir les logs de Tempo
kubectl logs -n opentelemetry deployment/tempo -f

# Vous devriez voir :
# level=info msg="received OTLP trace request"
# level=info msg="ingester flushing trace"
```

### 2. Tester l'envoi de traces

```bash
# Depuis un pod ou votre machine (avec port-forward)
curl -X POST http://tempo.opentelemetry:3200/api/echo

# Devrait retourner : "echo"
```

### 3. Vérifier les métriques générées

Dans Prometheus (http://localhost:9090), rechercher :

```promql
# Métriques de spans
tempo_spanmetrics_calls_total

# Latence par service
tempo_spanmetrics_latency_bucket{service_name="backend-service"}

# Service graph
traces_service_graph_request_total
```

### 4. Vérifier dans Grafana

1. Ouvrir Grafana : http://localhost:3000
2. **Explore** → **Tempo**
3. Faire une recherche par service name
4. Vous devriez voir vos traces avec tous les spans !

## 🐛 Troubleshooting

### Pas de traces visibles

1. **Vérifier que l'application envoie des traces** :
```bash
kubectl logs -n weeding-app deployment/backend-service | grep -i "otel\|trace"
```

2. **Vérifier que le collector reçoit les traces** :
```bash
kubectl logs -n opentelemetry deployment/otel-collector -f | grep -i trace
```

3. **Vérifier que Tempo reçoit du collector** :
```bash
kubectl logs -n opentelemetry deployment/tempo -f | grep -i "received\|ingesting"
```

### Traces incomplètes (spans manquants)

➡️ **Cause** : `trace_idle_period` trop court ou traces trop longues.

➡️ **Solution** : Augmenter dans tempo.yaml :
```yaml
ingester:
  trace_idle_period: 30s      # Attendre 30s au lieu de 20s

overrides:
  defaults:
    metrics_generator:
      max_trace_duration: 10m  # Traces jusqu'à 10min
```

### "Max trace size exceeded"

➡️ **Solution** : Augmenter les limites :
```yaml
overrides:
  defaults:
    ingestion:
      max_bytes_per_trace: 10_000_000    # 10 MB
      max_spans_per_trace: 200_000       # 200k spans
```

## 📚 Ressources

- [Tempo Configuration](https://grafana.com/docs/tempo/latest/configuration/)
- [Tempo Metrics Generator](https://grafana.com/docs/tempo/latest/metrics-generator/)
- [Grafana Tempo Datasource](https://grafana.com/docs/grafana/latest/datasources/tempo/)
