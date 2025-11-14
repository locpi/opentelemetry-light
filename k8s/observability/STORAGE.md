# Configuration Stockage Persistant - Rétention 15 Jours

## 📦 Vue d'Ensemble

La stack d'observabilité utilise maintenant des volumes persistants (PVC) pour stocker les données avec une rétention de 15 jours.

### Répartition des Volumes (Total : 50 GB)

| Composant   | Stockage | Rétention | Type de Données |
|-------------|----------|-----------|-----------------|
| Prometheus  | 15 GB    | 15 jours  | Métriques       |
| Loki        | 15 GB    | 15 jours  | Logs            |
| Tempo       | 15 GB    | 15 jours  | Traces          |
| Grafana     | 5 GB     | Permanent | Dashboards/Config |
| **TOTAL**   | **50 GB**| -         | -               |

## 🔧 Configuration par Composant

### Prometheus (Métriques - 15 GB)

**Rétention configurée** :
```yaml
args:
  - '--storage.tsdb.retention.time=15d'    # 15 jours
  - '--storage.tsdb.retention.size=14GB'   # Max 14GB (marge de sécurité)
```

**Volume** :
```yaml
volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: prometheus-storage  # 15 GB
```

**Ce qui est conservé 15 jours** :
- Métriques système (CPU, RAM, disque)
- Métriques JVM (heap, threads, GC)
- Métriques HTTP (requests, latency, errors)
- Métriques custom de vos applications
- Métriques générées par Tempo (span metrics, service graphs)

### Loki (Logs - 15 GB)

**Rétention configurée** :
```yaml
limits_config:
  retention_period: 360h  # 15 jours (360 heures)

table_manager:
  retention_deletes_enabled: true
  retention_period: 360h

compactor:
  retention_enabled: true
  retention_delete_delay: 2h
```

**Volume** :
```yaml
volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: loki-storage  # 15 GB
```

**Ce qui est conservé 15 jours** :
- Logs applicatifs de tous vos services
- Logs système Kubernetes
- Logs avec correlation trace_id
- Logs d'erreurs et exceptions

### Tempo (Traces - 15 GB)

**Rétention configurée** :
```yaml
compactor:
  compaction:
    block_retention: 360h  # 15 jours (360 heures)
```

**Volume** :
```yaml
volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: tempo-storage  # 15 GB
```

**Ce qui est conservé 15 jours** :
- Traces complètes avec tous les spans
- Traces des appels entre microservices
- Traces des requêtes BDD
- Traces des appels HTTP
- Métadonnées Kubernetes associées

### Grafana (Configuration - 5 GB)

**Volume** :
```yaml
volumes:
  - name: grafana-storage
    persistentVolumeClaim:
      claimName: grafana-storage  # 5 GB
```

**Ce qui est conservé de manière permanente** :
- Dashboards créés ou importés
- Configuration des datasources
- Utilisateurs et permissions
- Alertes configurées
- Annotations

## 🚀 Déploiement

### 1. Créer les PersistentVolumeClaims

```bash
# Créer les PVC (doit être fait en premier)
kubectl apply -f k8s/observability/00-storage.yaml

# Vérifier les PVC
kubectl get pvc -n opentelemetry
```

Vous devriez voir :
```
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES
prometheus-storage    Bound    pvc-xxx-xxx-xxx                           15Gi       RWO
loki-storage         Bound    pvc-yyy-yyy-yyy                           15Gi       RWO
tempo-storage        Bound    pvc-zzz-zzz-zzz                           15Gi       RWO
grafana-storage      Bound    pvc-www-www-www                           5Gi        RWO
```

### 2. Déployer la Stack avec Stockage Persistant

```bash
# Déployer tous les composants
kubectl apply -f k8s/observability/

# Vérifier que les pods utilisent les volumes
kubectl get pods -n opentelemetry
```

### 3. Vérifier les Montages de Volumes

```bash
# Pour Prometheus
kubectl describe pod -n opentelemetry -l app=prometheus | grep -A 5 "Volumes:"

# Pour Loki
kubectl describe pod -n opentelemetry -l app=loki | grep -A 5 "Volumes:"

# Pour Tempo
kubectl describe pod -n opentelemetry -l app=tempo | grep -A 5 "Volumes:"

# Pour Grafana
kubectl describe pod -n opentelemetry -l app=grafana | grep -A 5 "Volumes:"
```

## 📊 Estimation de l'Utilisation du Stockage

### Prometheus (15 GB)

Pour un cluster avec **5 services** instrumentés :

| Métrique | Estimation |
|----------|------------|
| Samples/sec | ~10,000 |
| Taille par sample | ~2 bytes |
| Par jour | ~1.7 GB |
| **15 jours** | **~25 GB** ⚠️ |

**Recommandation** : Avec 15 GB, vous pouvez conserver environ **8-9 jours** de métriques.

**Solutions** :
1. Augmenter le PVC à 30 GB
2. Réduire `scrape_interval` (actuellement 15s → 30s)
3. Activer le downsampling

### Loki (15 GB)

Pour un cluster avec **5 services** générant des logs :

| Métrique | Estimation |
|----------|------------|
| Logs/sec | ~100 |
| Taille moyenne | ~500 bytes |
| Par jour | ~4 GB (compressé: ~800 MB) |
| **15 jours** | **~12 GB** ✅ |

**Capacité suffisante** pour 15 jours avec compression.

### Tempo (15 GB)

Pour un cluster avec **5 services** avec traces :

| Métrique | Estimation |
|----------|------------|
| Traces/sec | ~10 |
| Spans par trace | ~20 |
| Taille par span | ~2 KB |
| Par jour | ~3.5 GB (compressé: ~700 MB) |
| **15 jours** | **~10 GB** ✅ |

**Capacité suffisante** pour 15 jours.

### Grafana (5 GB)

| Contenu | Estimation |
|---------|------------|
| Dashboards | ~50 MB |
| Configuration | ~10 MB |
| Utilisateurs | ~1 MB |
| **Total** | **~100 MB** ✅ |

**Capacité largement suffisante**.

## 📈 Ajustements Recommandés

### Si Prometheus Manque d'Espace

#### Option 1 : Augmenter le Volume

```yaml
# Dans 00-storage.yaml
resources:
  requests:
    storage: 30Gi  # Au lieu de 15Gi
```

Puis :
```bash
kubectl delete pvc prometheus-storage -n opentelemetry
kubectl apply -f k8s/observability/00-storage.yaml
kubectl rollout restart deployment/prometheus -n opentelemetry
```

#### Option 2 : Réduire l'Intervalle de Scraping

```yaml
# Dans 01-prometheus.yaml
global:
  scrape_interval: 30s  # Au lieu de 15s
```

**Impact** : Réduit de 50% la consommation de stockage mais moins de granularité.

#### Option 3 : Réduire la Rétention

```yaml
args:
  - '--storage.tsdb.retention.time=10d'  # Au lieu de 15d
```

### Si Loki Manque d'Espace

```yaml
# Dans 02-loki.yaml
limits_config:
  retention_period: 240h  # 10 jours au lieu de 15
```

### Si Tempo Manque d'Espace

```yaml
# Dans 03-tempo.yaml
compactor:
  compaction:
    block_retention: 240h  # 10 jours au lieu de 15
```

## 🔍 Monitoring de l'Espace Disque

### Vérifier l'Utilisation des PVC

```bash
# Script pour vérifier l'utilisation
for pod in $(kubectl get pods -n opentelemetry -o name); do
  echo "=== $pod ==="
  kubectl exec -n opentelemetry $pod -- df -h 2>/dev/null | grep -E '/prometheus|/loki|/var/tempo|/var/lib/grafana'
done
```

### Créer des Alertes Prometheus

Ajoutez dans votre configuration Prometheus :

```yaml
# Alerte si le disque est à 80%
- alert: HighDiskUsage
  expr: (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 > 80
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Disk usage is above 80%"
```

### Dashboard Grafana pour le Stockage

Créer un dashboard avec les requêtes :

```promql
# Utilisation Prometheus TSDB
prometheus_tsdb_storage_blocks_bytes / 1024 / 1024 / 1024

# Nombre de séries Prometheus
prometheus_tsdb_head_series

# Taille des blocs Tempo
tempo_ingester_blocks_total
```

## 🗑️ Nettoyage Manuel (si nécessaire)

### Forcer le Compactage Prometheus

```bash
kubectl exec -n opentelemetry deployment/prometheus -- \
  curl -X POST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones
```

### Forcer le Compactage Loki

```bash
# Redémarrer le compactor
kubectl rollout restart deployment/loki -n opentelemetry
```

### Forcer le Compactage Tempo

```bash
# Redémarrer Tempo
kubectl rollout restart deployment/tempo -n opentelemetry
```

## 🔄 Migration depuis emptyDir

Si vous avez déjà des données dans des volumes `emptyDir` :

### Sauvegarde des Données Grafana (Dashboards)

```bash
# Exporter les dashboards existants
kubectl port-forward -n opentelemetry svc/grafana 3000:3000 &
curl -H "Content-Type: application/json" \
  http://admin:admin@localhost:3000/api/search?type=dash-db \
  > dashboards-backup.json
```

### Restauration après Migration

Les données Prometheus, Loki et Tempo ne sont pas migrables facilement.
**Recommandation** : Accepter la perte des données historiques et redémarrer avec les volumes persistants.

## 📝 Backup et Restauration

### Backup des PVC (Recommandé)

```bash
# Installer Velero pour les backups
kubectl apply -f https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz

# Backup de tous les PVC
velero backup create observability-backup \
  --include-namespaces opentelemetry \
  --include-resources pvc,pv
```

### Snapshots Réguliers (si supporté par votre cloud provider)

Pour AWS EBS, GCP Persistent Disk, ou Azure Disk :

```yaml
# VolumeSnapshot (exemple)
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: prometheus-snapshot
  namespace: opentelemetry
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: prometheus-storage
```

## ⚠️ Limitations et Considérations

### Performances I/O

Les PVC sur Minikube utilisent le disque local :
- **Performances** : Dépendent du disque hôte
- **IOPS** : Limités par le storage backend

### Haute Disponibilité

Configuration actuelle :
- ❌ Pas de réplication
- ❌ Pas de backup automatique
- ❌ Single point of failure

Pour la production :
- ✅ Utiliser des StatefulSets avec réplication
- ✅ Configurer des backups automatiques
- ✅ Utiliser un storage class avec réplication (EBS, GCE PD)

### Croissance du Stockage

Surveillez régulièrement :
```bash
# Vérifier la croissance
kubectl exec -n opentelemetry deployment/prometheus -- \
  du -sh /prometheus

kubectl exec -n opentelemetry deployment/loki -- \
  du -sh /loki

kubectl exec -n opentelemetry deployment/tempo -- \
  du -sh /var/tempo
```

## 🎯 Résumé

✅ **Configuré** :
- 50 GB de stockage persistant répartis intelligemment
- Rétention de 15 jours pour métriques, logs et traces
- Conservation permanente des dashboards Grafana
- Données persistantes même en cas de redémarrage des pods

⚠️ **À Surveiller** :
- Utilisation du stockage Prometheus (peut atteindre la limite avant 15 jours)
- Croissance des données avec l'ajout de nouveaux services
- Performances I/O si beaucoup d'écritures

📊 **Prochaines Étapes** :
1. Déployer avec `kubectl apply -f k8s/observability/`
2. Monitorer l'utilisation du stockage
3. Ajuster les tailles de PVC si nécessaire
4. Configurer des backups réguliers
