# Architecture Globale - Analyse de Logs 5TB/Jour

## Vue d'ensemble

Cette architecture décrit une solution complète pour l'analyse de **5 TB de logs par jour** en utilisant Azure Data Explorer (ADX), une plateforme hautement optimisée pour l'analyse de données massives en temps réel.

## Contexte et Objectifs

### Contexte
- **Volume quotidien**: 5 TB de fichiers de logs
- **Besoin**: Analyse rapide, requêtes complexes, et visualisation
- **Échelle**: ~1.8 PB par an
- **Exigences**: Ingestion en temps quasi-réel, rétention flexible, performances élevées

### Objectifs de la Solution
1. **Performance**: Requêtes sub-secondes sur des milliards d'événements
2. **Scalabilité**: Supporter 5TB/jour avec possibilité d'évolution
3. **Disponibilité**: SLA 99.9% minimum
4. **Coût-efficacité**: Optimisation des coûts de stockage et compute
5. **Facilité d'utilisation**: Interface KQL intuitive pour les analystes

## Composants Principaux

### 1. Azure Data Explorer (ADX) - Cœur de la Solution

**Caractéristiques clés**:
- Moteur de requête distribué optimisé pour l'analyse de logs
- Compression native (~10:1 en moyenne)
- Indexation automatique de toutes les colonnes
- Support natif du format JSON, CSV, Parquet, Avro

**Configuration recommandée**:
- **Cluster**: Type Engine Standard_E16s_v5+
- **Nombre de nœuds**: 6-10 nœuds (évolutif selon charge)
- **Capacité d'ingestion**: ~200-250 GB/heure/nœud
- **Cache SSD**: Données chaudes (7-30 jours)
- **Stockage froid**: Azure Blob Storage pour archives

### 2. Couche d'Ingestion

**Options d'ingestion**:

#### Option A: Azure Event Hubs (Recommandé pour streaming)
- **Débit**: Jusqu'à 1 GB/sec par partition
- **Partitions**: 32 partitions minimum pour 5TB/jour
- **Rétention**: 1-7 jours dans Event Hub
- **Avantages**: Faible latence, découplage, buffer automatique

#### Option B: Azure Storage (Batch)
- **Pattern**: Drop de fichiers dans Blob Storage
- **Event Grid**: Notification automatique vers ADX
- **Format**: Compressed JSON, Parquet recommandé
- **Avantages**: Simple, économique pour batch

#### Option C: Hybrid
- Streaming pour logs critiques via Event Hubs
- Batch pour logs historiques via Storage

### 3. Transformation et Enrichissement

**Update Policy dans ADX**:
```kql
// Politique de mise à jour pour enrichissement automatique
.create-or-alter function EnrichLogs() {
    RawLogs
    | extend 
        ParsedTimestamp = todatetime(timestamp),
        Severity = case(
            level == "error", "High",
            level == "warning", "Medium",
            "Low"
        ),
        Region = extract(@"region=([^,]+)", 1, metadata)
    | project-away raw_field
}
```

### 4. Stockage Hiérarchisé

**Stratégie de rétention**:
- **Hot Cache (SSD)**: 7-14 jours - Requêtes ultra-rapides
- **Hot Storage**: 90 jours - Performance optimale
- **Cold Storage**: 1-2 ans - Coût réduit, accès occasionnel
- **Archive**: > 2 ans - Export vers Azure Archive Storage

### 5. Sécurité et Conformité

**Contrôles d'accès**:
- **Azure AD Integration**: Authentification centralisée
- **Row Level Security (RLS)**: Isolation par tenant/client
- **Chiffrement**: At-rest (Azure Storage Encryption) et in-transit (TLS 1.2+)
- **Audit**: Logs d'audit ADX activés

**Compliance**:
- GDPR: Support PII masking et data retention policies
- SOC 2, ISO 27001 compliance native Azure

## Architecture de Déploiement

### Configuration Multi-régions

**Région primaire** (ex: West Europe):
- Cluster ADX primaire
- Event Hubs namespace primaire
- Compute pour ingestion

**Région secondaire** (ex: North Europe):
- Cluster ADX en standby ou lecture
- Réplication via Follower Database
- Disaster Recovery: RPO < 1h, RTO < 4h

### Réseau et Connectivité

- **Virtual Network**: Intégration VNet pour isolation
- **Private Endpoints**: Connexions privées pour ADX
- **Service Endpoints**: Event Hubs, Storage sécurisés
- **Firewall**: IP whitelisting sur cluster ADX

## Flux de Données

1. **Génération de logs** → Applications/Services
2. **Collection** → Agents (Fluentd, Logstash, Azure Monitor Agent)
3. **Streaming** → Event Hubs (buffering)
4. **Ingestion** → ADX (compression, indexation)
5. **Transformation** → Update policies (enrichissement)
6. **Stockage** → Tiered storage (hot → warm → cold)
7. **Analyse** → KQL queries via portails/APIs
8. **Visualisation** → Power BI, Grafana, Azure Dashboards

## Patterns d'Usage

### Use Case 1: Monitoring Temps Réel
- Dashboards en temps réel (rafraîchissement 30s)
- Alertes automatiques sur anomalies
- Latence: < 2 minutes end-to-end

### Use Case 2: Investigations de Sécurité
- Requêtes ad-hoc sur 90 jours de données
- Corrélation d'événements complexes
- Time to insight: Secondes à minutes

### Use Case 3: Analytics Historiques
- Tendances mensuelles/annuelles
- Capacité planning
- Requêtes sur cold storage: Minutes

## Intégrations

### Sources de Logs
- **Azure**: Application Insights, Azure Monitor, Activity Logs
- **On-premises**: Syslog, Windows Events via agents
- **Applications**: Logs applicatifs JSON/structurés
- **Sécurité**: Firewall logs, IDS/IPS, WAF

### Consommateurs
- **Power BI**: Dashboards exécutifs
- **Grafana**: Dashboards opérationnels
- **Azure Logic Apps**: Automatisation basée sur requêtes
- **APIs REST**: Intégration applications custom
- **ADX Web UI**: Exploration interactive

## Évolutivité

### Scaling Vertical
- Augmentation SKU des nœuds (E8 → E16 → E32 → E64)
- Impact: Plus de RAM, CPU pour requêtes complexes

### Scaling Horizontal
- Ajout de nœuds au cluster
- Impact: Plus de débit d'ingestion et parallélisme des requêtes

### Auto-scaling
- Configuration basée sur:
  - CPU utilization (> 70%)
  - Ingestion queue depth
  - Query latency metrics

## Monitoring et Observabilité

### Métriques clés
- **Ingestion**:
  - Events ingested/sec
  - Ingestion latency
  - Failed ingestions
  
- **Query Performance**:
  - Query duration (p50, p95, p99)
  - Queries per second
  - Cache hit ratio

- **Ressources**:
  - CPU/Memory utilization
  - Disk IOPS
  - Network throughput

### Outils de Monitoring
- **Azure Monitor**: Métriques et alertes
- **ADX Insights**: Dashboard intégré
- **Log Analytics**: Meta-monitoring des logs ADX
- **Application Insights**: Pour applications connectées

## Haute Disponibilité

### Résilience du Cluster
- **Nœuds multiples**: Minimum 3 nœuds pour HA
- **Réplication**: Data repliquée sur multiple nœuds
- **Zone Redundancy**: Distribution sur Availability Zones

### Disaster Recovery
- **Follower Databases**: Lecture seule en région secondaire
- **Backup**: Continuous export vers Storage
- **Recovery**: Restore from Storage ou promote follower

## Optimisations

### Performance
1. **Partitioning**: Par date (colonnes datetime)
2. **Materialized Views**: Pré-agrégations pour requêtes fréquentes
3. **Cache Policy**: Optimisation du hot cache
4. **Extent Management**: Merge policy pour petits extents

### Coûts
1. **Tiered Storage**: Données froides en Azure Storage
2. **Reserved Capacity**: Réservations 1-3 ans
3. **Continuous Export**: Archive vers Blob Storage
4. **Auto-pause**: Dev/Test clusters en dehors heures

## Conformité et Gouvernance

### Data Governance
- **Data Classification**: Tags sur tables/colonnes
- **Retention Policies**: Automatiques par table
- **Data Lineage**: Tracking via metadata

### Audit et Logging
- **Audit Logs**: Toutes opérations DDL/DML
- **Query Audit**: Tracking des queries utilisateurs
- **Access Logs**: Connexions et authentifications

## Roadmap et Évolution

### Phase 1 (Mois 1-2): Foundation
- Déploiement cluster ADX de base
- Ingestion batch via Storage
- Requêtes KQL basiques

### Phase 2 (Mois 3-4): Optimisation
- Migration vers streaming Event Hubs
- Update policies pour enrichissement
- Dashboards Power BI/Grafana

### Phase 3 (Mois 5-6): Scaling
- Multi-region setup
- Optimisations performance avancées
- Intégrations additionnelles

### Phase 4 (Mois 6+): Excellence Opérationnelle
- Auto-scaling automation
- ML pour détection anomalies
- Self-service analytics pour équipes

## Références

- [Azure Data Explorer Documentation](https://docs.microsoft.com/azure/data-explorer/)
- [KQL Quick Reference](https://docs.microsoft.com/azure/data-explorer/kql-quick-reference)
- [ADX Best Practices](https://docs.microsoft.com/azure/data-explorer/best-practices)
- [Pricing Calculator](https://azure.microsoft.com/pricing/calculator/)

## Documents Connexes

- [Solution Technique ADX](./adx-solution.md)
- [Exemples KQL](./kql-examples.md)
- [Stratégie d'Ingestion](./data-ingestion.md)
- [Estimation de Coûts](./cost-estimation.md)
- [Diagrammes d'Architecture](./diagrams/)
