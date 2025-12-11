# LogAnalysis-5TB-Day

## Vue d'ensemble

Ce repository contient l'architecture et la documentation pour une solution d'analyse de **5 TB de logs par jour** en utilisant **Azure Data Explorer (ADX)** avec le langage de requête **KQL (Kusto Query Language)**.

## 📋 Contenu de la Documentation

### 📐 Architecture

#### [Architecture Globale](./architecture.md)
Document principal décrivant l'architecture complète de la solution:
- Vue d'ensemble et objectifs
- Composants principaux (ADX, Event Hubs, Storage)
- Stratégies de déploiement et sécurité
- Haute disponibilité et disaster recovery
- Monitoring et optimisations

#### [Solution Technique ADX](./adx-solution.md)
Détails techniques approfondis sur Azure Data Explorer:
- Pourquoi ADX pour l'analyse de logs
- Architecture détaillée du cluster
- Configuration et sizing (8-10 nœuds E16s_v5)
- Schémas de tables optimisés
- Materialized views et update policies
- Best practices et cas d'usage réels

### 🔄 Ingestion de Données

#### [Stratégie d'Ingestion](./data-ingestion.md)
Guide complet sur l'ingestion de 5TB/jour:
- **Option 1**: Azure Event Hubs (Streaming) - Recommandé
  - Latence < 2 minutes
  - Configuration 32+ partitions
- **Option 2**: Azure Blob Storage (Batch)
  - Coût réduit
  - Latence 5-15 minutes
- **Option 3**: Hybrid (Streaming + Batch)
- Transformation et enrichissement avec Update Policies
- Monitoring et troubleshooting

### 📊 Requêtes et Analyse

#### [Exemples KQL](./kql-examples.md)
Collection complète de requêtes KQL pour l'analyse de logs:
- Analyses de base (comptages, filtres)
- Analyse temporelle et time series
- Performance et latence
- Analyse d'erreurs et troubleshooting
- Sécurité et audit
- Distributed tracing multi-service
- Analytics business
- Monitoring et alerting
- Fonctions personnalisées et best practices

### 💰 Estimation Financière

#### [Estimation de Coûts](./cost-estimation.md)
Analyse détaillée des coûts (Step 2):
- **Coût mensuel baseline**: $15,000-20,000 (~$180,000-240,000/an)
- **Coût optimisé avec RI**: $13,000-15,000 (~$156,000-180,000/an)
- Détail par composant (ADX, Event Hubs, Storage, Network)
- 3 scénarios: Production Optimisée, Startup, Enterprise HA
- Stratégies d'optimisation (Reserved Instances, Tiered Storage)
- Comparaison avec alternatives (Elasticsearch, Splunk, Datadog)
- ROI et justification business

### 🎨 Diagrammes d'Architecture

#### [Diagrammes PUML](./diagrams/)
Diagrammes PlantUML pour visualiser l'architecture:

1. **[architecture-overview.puml](./diagrams/architecture-overview.puml)**
   - Vue d'ensemble complète de la solution
   - Sources → Collection → Ingestion → ADX → Consommation
   - Sécurité, monitoring, DR

2. **[data-flow.puml](./diagrams/data-flow.puml)**
   - Flux de données détaillé end-to-end
   - Séquence d'ingestion avec timings
   - 5 étapes: Génération → Collection → Transport → Ingestion → Query

3. **[adx-cluster.puml](./diagrams/adx-cluster.puml)**
   - Architecture interne du cluster ADX
   - Engine nodes, storage tiers, data management
   - Spécifications techniques par nœud

**Pour générer les images PNG** depuis les fichiers PUML:
```bash
# Installation PlantUML
brew install plantuml  # macOS
# ou
sudo apt install plantuml  # Ubuntu

# Génération des diagrammes
plantuml diagrams/*.puml
```

## 🎯 Solution Proposée

### Architecture Recommandée

**Cluster Azure Data Explorer**:
- **8 nœuds** Standard_E16s_v5 (16 cores, 128GB RAM, 512GB SSD)
- **Capacité d'ingestion**: 1,600 GB/h (marge pour peaks 3x)
- **Hot cache**: 4 TB SSD (7-14 jours)
- **Hot storage**: 90 jours (~4.5 TB compressé)
- **Cold storage**: 1-2 ans (Azure Blob Cool Tier)

**Ingestion via Event Hubs**:
- **32 partitions** pour parallélisme
- **10-20 Throughput Units** avec auto-inflate
- **Latence**: < 2 minutes end-to-end
- **Format**: JSON compressé (GZip) ou Parquet

**Performance**:
- **Requêtes**: Sub-seconde sur milliards d'événements
- **Compression**: Ratio 10:1 (5TB → 500GB stocké)
- **SLA**: 99.9% (99.99% avec multi-région)

### Métriques Clés

| Métrique | Valeur |
|----------|--------|
| Volume quotidien | 5 TB |
| Débit moyen | 208 GB/h (~58 MB/s) |
| Débit peak (3x) | 625 GB/h |
| Latence ingestion | < 2 minutes |
| Latence query (hot cache) | < 500ms |
| Compression ratio | 10:1 |
| Rétention hot | 90 jours |
| Rétention cold | 1-2 ans |

### Coûts Estimés

| Scénario | Mensuel | Annuel |
|----------|--------:|-------:|
| **Baseline (sans optimisations)** | $20,340 | $244,080 |
| **Production Standard** | $15,291 | $183,492 |
| **Optimisé (RI 3 ans)** | $13,465 | $161,580 |
| **Startup Minimal** | $5,000 | $60,000 |
| **Enterprise HA Multi-région** | $17,273 | $207,276 |

## 🚀 Démarrage Rapide

### Prérequis
- Subscription Azure
- Azure CLI installé
- Permissions pour créer ressources (Contributor role minimum)

### Étape 1: Créer le Cluster ADX

```bash
# Variables
RG="rg-loganalysis-prod"
LOCATION="westeurope"
CLUSTER_NAME="loganalysis-prod-adx"

# Créer resource group
az group create --name $RG --location $LOCATION

# Créer cluster ADX
az kusto cluster create \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_E16s_v5 \
  --capacity 8 \
  --enable-streaming-ingest true \
  --zones "1,2,3"
```

### Étape 2: Créer Event Hub

```bash
# Event Hub Namespace
az eventhubs namespace create \
  --name "loganalysis-prod-eh" \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard \
  --enable-auto-inflate true \
  --maximum-throughput-units 20

# Event Hub
az eventhubs eventhub create \
  --name "application-logs" \
  --namespace-name "loganalysis-prod-eh" \
  --partition-count 32 \
  --message-retention 1
```

### Étape 3: Configurer ADX Database et Tables

Se référer à [adx-solution.md](./adx-solution.md) section "Configuration Détaillée" pour:
- Création de database
- Définition du schéma de table
- Ingestion mapping
- Data connection vers Event Hub

### Étape 4: Déployer Agents de Collecte

Se référer à [data-ingestion.md](./data-ingestion.md) section "Configuration des Agents" pour:
- Fluentd configuration
- Vector configuration
- Routage et buffering

## 📚 Ressources

### Documentation Officielle
- [Azure Data Explorer](https://docs.microsoft.com/azure/data-explorer/)
- [KQL Reference](https://docs.microsoft.com/azure/data-explorer/kql-quick-reference)
- [Event Hubs Documentation](https://docs.microsoft.com/azure/event-hubs/)

### Outils
- [ADX Web UI](https://dataexplorer.azure.com/)
- [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/)
- [KQL Playground](https://dataexplorer.azure.com/clusters/help/databases/Samples)

### Tutoriels
- [ADX Best Practices](https://docs.microsoft.com/azure/data-explorer/best-practices)
- [Ingestion Best Practices](https://docs.microsoft.com/azure/data-explorer/ingest-data-overview)
- [Query Best Practices](https://docs.microsoft.com/azure/data-explorer/kusto/query/best-practices)

## 🤝 Contributions

Ce repository est un document d'architecture. Pour des questions ou suggestions:
- Ouvrir une issue
- Proposer des améliorations via PR
- Contacter l'équipe Platform Engineering

## 📝 Licence

Documentation sous licence MIT - voir fichier LICENSE

## ✅ Checklist de Mise en Œuvre

### Phase 1: Foundation (Mois 1-2)
- [ ] Provisionner cluster ADX
- [ ] Configurer Event Hub ou Blob Storage
- [ ] Créer databases et tables
- [ ] Setup ingestion mappings
- [ ] Tester ingestion end-to-end
- [ ] Configurer monitoring de base

### Phase 2: Optimisation (Mois 3-4)
- [ ] Implémenter Update Policies
- [ ] Créer Materialized Views
- [ ] Optimiser hot cache policy
- [ ] Setup dashboards (Power BI/Grafana)
- [ ] Configurer alertes critiques
- [ ] Documenter runbooks

### Phase 3: Scaling (Mois 5-6)
- [ ] Acheter Reserved Instances
- [ ] Implémenter tiered storage
- [ ] Setup follower database (DR)
- [ ] Optimiser query performance
- [ ] Activer auto-scaling
- [ ] Audit de coûts

### Phase 4: Excellence (Mois 7+)
- [ ] ML pour anomaly detection
- [ ] Self-service analytics
- [ ] Advanced security (RLS)
- [ ] Multi-tenant isolation
- [ ] Continuous optimization

## 📊 Statut du Projet

**Step 1**: ✅ **Complété** - Documentation Azure Data Explorer avec KQL
- Architecture globale
- Solution technique détaillée
- Exemples KQL complets
- Stratégie d'ingestion
- Diagrammes PUML

**Step 2**: ✅ **Complété** - Estimation financière
- Coûts détaillés par composant
- 3 scénarios (Baseline, Optimisé, Enterprise)
- Comparaison avec alternatives
- Stratégies d'optimisation
- ROI et justification

---

**Version**: 1.0  
**Date**: Juin 2024  
**Auteur**: Architecture Team