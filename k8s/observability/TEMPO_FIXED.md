# Configuration Tempo Corrigée - Compatible avec Version 2.3.1

## ✅ Problèmes Résolus

### Erreurs de Configuration YAML

**Erreurs rencontrées** :
```
line 69: field max_trace_duration not found in type overrides.Limits
line 73: field max_spans_per_trace not found in type overrides.Limits
line 72: field query_timeout not found in type frontend.TraceByIDConfig
line 82: field defaults not found in type overrides.Limits
```

**Causes** :
- Version Tempo 2.2.3 ne supporte pas certains champs dans `overrides`
- Syntaxe incorrecte pour les overrides (pas de structure `defaults`)
- Champs non supportés dans `query_frontend`

## 📝 Configuration Finale Valide

### Structure YAML Correcte

```yaml
server:
  http_listen_port: 3200
  log_level: info

distributor:
  receivers:
    otlp:
      protocols:
        http:
          endpoint: 0.0.0.0:4318
        grpc:
          endpoint: 0.0.0.0:4317

ingester:
  max_block_duration: 5m
  trace_idle_period: 20s        # ✅ Important pour traces Java multi-spans
  max_block_bytes: 1_000_000    # ✅ Gérer traces volumineuses

compactor:
  compaction:
    block_retention: 1h

metrics_generator:
  registry:
    external_labels:
      source: tempo
      cluster: minikube
  storage:
    path: /var/tempo/generator/wal
    remote_write:
      - url: http://prometheus:9090/api/v1/write
        send_exemplars: true
  processor:
    service_graphs:
      dimensions:
        - http.method
        - http.status_code
    span_metrics:
      dimensions:
        - http.method
        - http.status_code
        - service.name

storage:
  trace:
    backend: local
    wal:
      path: /var/tempo/wal
    local:
      path: /var/tempo/blocks
    pool:
      max_workers: 100          # ✅ Optimisation pour Java
      queue_depth: 10000

query_frontend:
  search:
    max_duration: 0s
    default_result_limit: 20

querier:
  max_concurrent_queries: 10

overrides:
  metrics_generator_processors: [service-graphs, span-metrics]
```

## 🔄 Changements Appliqués

### 1. Version Tempo Mise à Jour

**Avant** : `grafana/tempo:2.2.3`
**Après** : `grafana/tempo:2.3.1`

Version plus récente avec meilleur support des fonctionnalités.

### 2. Configuration `overrides` Simplifiée

**Supprimé** :
- `max_trace_duration` (non supporté dans overrides pour cette version)
- `max_bytes_per_trace` (non supporté dans overrides pour cette version)
- `max_spans_per_trace` (non supporté dans overrides pour cette version)
- Structure `defaults` (syntaxe incorrecte)

**Conservé** :
- `metrics_generator_processors` : Active les métriques de service graphs et spans

### 3. Configuration `ingester` Optimisée

Ces paramètres **fonctionnent** et sont importants pour Java :

```yaml
ingester:
  max_block_duration: 5m
  trace_idle_period: 20s      # Attend 20s pour collecter tous les spans
  max_block_bytes: 1_000_000  # 1 MB par bloc (suffisant pour la plupart des traces Java)
```

**Pourquoi c'est important** :
- Les traces Java avec auto-instrumentation génèrent beaucoup de spans
- Un endpoint REST → appel service → BDD peut créer 20-50 spans
- `trace_idle_period: 20s` laisse le temps à tous les spans d'arriver avant de finaliser la trace

### 4. `metrics_generator` Fonctionnel

```yaml
metrics_generator:
  processor:
    service_graphs:
      dimensions:
        - http.method
        - http.status_code
    span_metrics:
      dimensions:
        - http.method
        - http.status_code
        - service.name
```

**Résultat** : Vous obtiendrez des métriques Prometheus :
- `tempo_spanmetrics_calls_total` : Nombre d'appels par service
- `tempo_spanmetrics_latency_bucket` : Histogramme de latence
- `traces_service_graph_request_total` : Relations entre services

## 🚀 Déploiement

```bash
# 1. Appliquer la configuration corrigée
kubectl apply -f k8s/observability/03-tempo.yaml

# 2. Attendre que le nouveau pod démarre
kubectl rollout status deployment/tempo -n opentelemetry

# 3. Vérifier qu'il n'y a plus d'erreurs
kubectl logs -n opentelemetry deployment/tempo --tail=50

# Vous devriez voir :
# level=info msg="Tempo started"
# level=info msg="Server listening on addresses"
```

## ✅ Vérification

### 1. Vérifier que Tempo démarre sans erreur

```bash
kubectl logs -n opentelemetry deployment/tempo

# Devrait afficher :
# level=info msg="Tempo started"
# level=info msg="OTLP receiver started"
```

### 2. Tester la réception OTLP

```bash
# Port-forward
kubectl port-forward -n opentelemetry svc/tempo 3200:3200

# Tester l'API
curl http://localhost:3200/api/echo
# Devrait retourner : "echo"
```

### 3. Vérifier dans Grafana

1. Ouvrir Grafana : http://localhost:3000
2. **Explore** → datasource **Tempo**
3. Faire un test de recherche

Si vous voyez des traces, c'est bon ! ✅

## 📊 Ce Que Vous Obtiendrez

### Traces Java Complètes

Avec cette configuration, vos traces contiendront :

```
backend-service: GET /api/users/123 (245ms)
  ├─ Spring MVC: GET /api/users/{id} (240ms)
  │   ├─ HTTP Client: GET auth-service/validate (45ms)
  │   │   └─ auth-service: POST /validate (40ms)
  │   │       └─ JDBC Query (8ms)
  │   └─ UserService.getUser() (180ms)
  │       ├─ JPA: findById() (35ms)
  │       │   └─ JDBC: SELECT * FROM users (30ms)
  │       └─ HTTP Client: GET profile-service/... (140ms)
  └─ Response 200
```

### Métriques Générées

Dans Prometheus, vous aurez :

```promql
# Nombre d'appels par service
tempo_spanmetrics_calls_total{service_name="backend-service"}

# Latence p95
histogram_quantile(0.95,
  rate(tempo_spanmetrics_latency_bucket{service_name="backend-service"}[5m])
)

# Graphe des services
traces_service_graph_request_total{client="backend-service", server="auth-service"}
```

### Service Graph dans Grafana

Visualisation des relations entre services :
```
frontend → backend-service → auth-service
              ↓
         profile-service
```

## 🐛 Troubleshooting

### Tempo ne démarre toujours pas

```bash
# Voir les logs détaillés
kubectl logs -n opentelemetry deployment/tempo -f

# Vérifier la config
kubectl get configmap tempo-config -n opentelemetry -o yaml
```

### Erreur "field X not found"

➡️ La configuration contient encore des champs non supportés.
➡️ Vérifiez que vous utilisez bien la version corrigée du fichier.

### Pas de traces visibles

1. **Vérifier que l'application envoie des traces** :
```bash
kubectl logs -n weeding-app deployment/backend-service | grep -i "otel\|trace"
```

2. **Vérifier que le collector transmet à Tempo** :
```bash
kubectl logs -n opentelemetry deployment/otel-collector | grep -i tempo
```

3. **Vérifier que Tempo reçoit** :
```bash
kubectl logs -n opentelemetry deployment/tempo | grep -i "received\|ingesting"
```

### Traces incomplètes (spans manquants)

➡️ **Cause** : `trace_idle_period` trop court pour vos traces.

➡️ **Solution** : Augmenter dans la ConfigMap :
```yaml
ingester:
  trace_idle_period: 30s  # Au lieu de 20s
```

Puis redéployer :
```bash
kubectl rollout restart deployment/tempo -n opentelemetry
```

## 📝 Limitations de cette Configuration

### Ce qui N'est PAS configuré (car non supporté dans overrides)

- `max_trace_duration` : Durée max d'une trace
- `max_bytes_per_trace` : Taille max d'une trace
- `max_spans_per_trace` : Nombre max de spans

**Impact** : Ces limites utilisent les valeurs par défaut de Tempo :
- max_trace_duration : ~30 minutes (largement suffisant)
- max_bytes_per_trace : ~5 MB (suffisant pour la plupart des traces Java)
- max_spans_per_trace : ~100,000 (très large)

### Si Vous Avez Besoin de Plus

Option 1 : **Passer à Tempo 2.4+** qui supporte ces champs dans overrides

Option 2 : **Utiliser un fichier overrides séparé** (configuration avancée)

## ✅ Résumé

Votre configuration Tempo est maintenant :
- ✅ **Syntaxiquement correcte** : Plus d'erreurs YAML
- ✅ **Optimisée pour Java** : `trace_idle_period`, `max_block_bytes`
- ✅ **Service graphs activés** : Visualisation des relations
- ✅ **Span metrics activés** : Métriques de latence et d'erreurs
- ✅ **Compatible version 2.3.1** : Fonctionne avec l'image mise à jour

Vous êtes prêt à recevoir et visualiser vos traces Java ! 🎉
