# Stack d'Observabilité Kubernetes (Mode Léger)

Configuration Kubernetes légère pour déployer Grafana, Tempo, Loki et Prometheus sur Minikube.

## Composants

- **Prometheus** (v2.47.0) : Collecte de métriques
- **Loki** (v2.9.0) : Agrégation de logs
- **Tempo** (v2.2.3) : Tracing distribué
- **Grafana** (v10.1.5) : Visualisation et tableaux de bord

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
kubectl apply -f k8s/observability/
```

### 2. Vérifier le déploiement

```bash
# Vérifier que tous les pods sont en cours d'exécution
kubectl get pods -n observability

# Vérifier les services
kubectl get svc -n observability
```

### 3. Accéder à Grafana

```bash
# Port-forward pour accéder à Grafana
kubectl port-forward -n observability svc/grafana 3000:3000
```

Ouvrir http://localhost:3000 dans votre navigateur.

L'authentification anonyme est activée avec le rôle Admin.

## Accès aux Services

### Grafana
```bash
kubectl port-forward -n observability svc/grafana 3000:3000
# http://localhost:3000
```

### Prometheus
```bash
kubectl port-forward -n observability svc/prometheus 9090:9090
# http://localhost:9090
```

### Tempo (OTLP endpoint pour vos applications)
```bash
# OTLP gRPC
kubectl port-forward -n observability svc/tempo 4317:4317

# OTLP HTTP
kubectl port-forward -n observability svc/tempo 4318:4318
```

### Loki
```bash
kubectl port-forward -n observability svc/loki 3100:3100
# http://localhost:3100
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

## Envoyer des Traces à Tempo

### Variables d'environnement OTLP

Pour vos applications instrumentées avec OpenTelemetry :

```bash
# Depuis l'intérieur du cluster
OTEL_EXPORTER_OTLP_ENDPOINT=http://tempo.observability.svc.cluster.local:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# Ou pour gRPC
OTEL_EXPORTER_OTLP_ENDPOINT=http://tempo.observability.svc.cluster.local:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
```

### Annotations Prometheus pour vos Pods

Pour que Prometheus scrape les métriques de vos applications :

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

## Nettoyage

```bash
# Supprimer tous les composants
kubectl delete -f k8s/observability/

# Ou supprimer le namespace complet
kubectl delete namespace observability
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

## Ressources Utiles

- [Documentation Prometheus](https://prometheus.io/docs/)
- [Documentation Loki](https://grafana.com/docs/loki/latest/)
- [Documentation Tempo](https://grafana.com/docs/tempo/latest/)
- [Documentation Grafana](https://grafana.com/docs/grafana/latest/)
- [OpenTelemetry](https://opentelemetry.io/)
