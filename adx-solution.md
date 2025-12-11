# Solution Technique Azure Data Explorer (ADX)

## Introduction à Azure Data Explorer

Azure Data Explorer (ADX) est un service d'analytique de données rapide et hautement scalable, optimisé pour l'analyse ad-hoc de grands volumes de données structurées, semi-structurées et non structurées, notamment les logs et télémétries.

## Pourquoi ADX pour l'Analyse de Logs 5TB/Jour?

### Avantages Techniques

#### 1. Performance Exceptionnelle
- **Requêtes sub-secondes**: Scanning de milliards de lignes en < 1 seconde
- **Compression intelligente**: Ratio 10:1 à 50:1 selon les données
- **Indexation automatique**: Toutes colonnes indexées sans configuration
- **Cache hiérarchisé**: Hot cache SSD pour données fréquentes

#### 2. Ingestion Haute Performance
- **Débit**: 200+ GB/heure par nœud
- **Latence**: < 2 minutes pour données disponibles en requête
- **Batch et Streaming**: Support des deux modes
- **Formats multiples**: JSON, CSV, Parquet, Avro, ORC, PSV, TSV

#### 3. Langage KQL (Kusto Query Language)
- **Expressivité**: SQL-like mais optimisé pour logs
- **Opérateurs puissants**: render, summarize, join, mv-expand
- **Analyse temporelle**: Time series natives (make-series, series_decompose)
- **Machine Learning**: ML operators intégrés

#### 4. Coût-Efficacité
- **Compression élevée**: 5TB brut → ~500GB-1TB stocké
- **Tiered storage**: Hot (SSD) + Cold (Blob) automatique
- **Pay-per-use**: Scaling up/down selon besoin
- **Pas de licensing**: Inclus dans le coût Azure

## Architecture Technique ADX

### Composants du Cluster

#### 1. Engine Nodes (Compute)
**Responsabilités**:
- Exécution des requêtes
- Ingestion et indexation des données
- Gestion du cache local
- Compression et décompression

**Sizing pour 5TB/jour**:
```
SKU: Standard_E16s_v5 (16 cores, 128GB RAM, 512GB SSD cache)
Nombre de nœuds: 8-10 nœuds
Capacité totale cache: 4-5 TB (hot cache)
Débit ingestion: ~1.6-2 TB/heure (peak)
```

**Justification**:
- 5TB / 24h = ~208 GB/h en moyenne
- Peak factor 3x = ~625 GB/h
- Chaque nœud E16s = ~200GB/h
- 8 nœuds = 1.6TB/h (marge confortable)

#### 2. Data Management (Métadonnées)
- Catalogue de tables et schémas
- Extent management (data shards)
- Policies de rétention et cache
- Query orchestration

#### 3. Storage Layer
**Hot Storage (SSD)**:
- Données récentes (7-14 jours)
- Accès ultra-rapide (< 100ms)
- Taille: ~350-700 GB (5TB compressé à 10:1)

**Cold Storage (Azure Blob)**:
- Données plus anciennes (> 14 jours)
- Accès rapide (~1-3 secondes)
- Taille: ~18-36 TB pour 1 an de rétention
- Coût: ~$0.0184/GB/mois (LRS cool tier)

## Configuration Détaillée

### 1. Création du Cluster

```bash
# Via Azure CLI
az kusto cluster create \
  --cluster-name "loganalysis-prod-adx" \
  --resource-group "rg-loganalysis-prod" \
  --location "westeurope" \
  --sku "Standard_E16s_v5" \
  --capacity 8 \
  --enable-streaming-ingest true \
  --enable-purge true \
  --zones "1,2,3"
```

**Paramètres clés**:
- `enable-streaming-ingest`: Pour latence minimale (<30s)
- `enable-purge`: Pour conformité GDPR
- `zones`: HA avec distribution sur 3 availability zones

### 2. Création de la Database

```kql
.create database LogAnalyticsDB 

// Configuration de la rétention hot/cold
.alter database LogAnalyticsDB policy retention 
```json
{
  "SoftDeletePeriod": "365.00:00:00",
  "Recoverability": "Enabled"
}
```

.alter database LogAnalyticsDB policy caching hot = 14d
```

### 3. Schéma de Table Optimisé

```kql
.create-merge table ApplicationLogs (
    Timestamp: datetime,
    Level: string,
    Logger: string,
    Message: string,
    Exception: string,
    Properties: dynamic,
    TraceId: string,
    SpanId: string,
    ServiceName: string,
    Environment: string,
    HostName: string,
    Region: string,
    TenantId: string,
    UserId: string,
    SessionId: string,
    RequestPath: string,
    ResponseCode: int,
    DurationMs: long,
    BytesSent: long,
    BytesReceived: long
)

// Partitioning pour optimisation
.alter table ApplicationLogs policy partitioning 
```json
{
  "PartitionKeys": [
    {
      "ColumnName": "Timestamp",
      "Kind": "UniformRange",
      "Properties": {
        "Reference": "1970-01-01T00:00:00",
        "RangeSize": "1.00:00:00",
        "OverrideCreationTime": false
      }
    }
  ]
}
```

// Politique de rétention spécifique
.alter table ApplicationLogs policy retention 
```json
{
  "SoftDeletePeriod": "365.00:00:00"
}
```

// Merge policy pour optimiser les extents
.alter table ApplicationLogs policy merge 
```json
{
  "MaxRangeInHours": 8,
  "RowCountUpperBoundForMerge": 16000000
}
```
```

### 4. Ingestion Mapping

```kql
.create-or-alter table ApplicationLogs ingestion json mapping 'ApplicationLogsMapping'
```
[
  {"column": "Timestamp", "path": "$.timestamp", "datatype": "datetime"},
  {"column": "Level", "path": "$.level", "datatype": "string"},
  {"column": "Logger", "path": "$.logger", "datatype": "string"},
  {"column": "Message", "path": "$.message", "datatype": "string"},
  {"column": "Exception", "path": "$.exception", "datatype": "string"},
  {"column": "Properties", "path": "$.properties", "datatype": "dynamic"},
  {"column": "TraceId", "path": "$.traceId", "datatype": "string"},
  {"column": "SpanId", "path": "$.spanId", "datatype": "string"},
  {"column": "ServiceName", "path": "$.serviceName", "datatype": "string"},
  {"column": "Environment", "path": "$.environment", "datatype": "string"},
  {"column": "HostName", "path": "$.hostName", "datatype": "string"},
  {"column": "Region", "path": "$.region", "datatype": "string"}
]
```
```

## Stratégies d'Optimisation

### 1. Materialized Views

Pour requêtes fréquentes et agrégations coûteuses:

```kql
.create materialized-view ErrorSummary on table ApplicationLogs
{
    ApplicationLogs
    | where Level in ("Error", "Critical")
    | summarize 
        ErrorCount = count(),
        UniqueUsers = dcount(UserId),
        AffectedServices = dcount(ServiceName)
      by bin(Timestamp, 5m), Level, ServiceName
}

// Active backfill pour données historiques
.alter materialized-view ErrorSummary policy materialized_view_effectivedatetime datetime(2024-01-01)
```

**Bénéfices**:
- Requêtes 100-1000x plus rapides
- Réduction de compute
- Données pré-agrégées toujours à jour

### 2. Update Policies (Transformation)

Pour enrichissement automatique à l'ingestion:

```kql
// Table cible enrichie
.create table EnrichedApplicationLogs (
    Timestamp: datetime,
    Level: string,
    SeverityScore: int,
    IsError: bool,
    ServiceCategory: string,
    GeoLocation: dynamic,
    Message: string,
    [...]
)

// Fonction d'enrichissement
.create-or-alter function EnrichLogEntry() {
    ApplicationLogs
    | extend 
        SeverityScore = case(
            Level == "Critical", 5,
            Level == "Error", 4,
            Level == "Warning", 3,
            Level == "Info", 2,
            1
        ),
        IsError = Level in ("Error", "Critical"),
        ServiceCategory = extract(@"^([^-]+)", 1, ServiceName),
        GeoLocation = geo_info_from_ip_address(extract(@"ip=([0-9.]+)", 1, Message))
}

// Update policy
.alter table EnrichedApplicationLogs policy update 
@'[{"Source": "ApplicationLogs", "Query": "EnrichLogEntry()", "IsEnabled": true, "IsTransactional": true}]'
```

### 3. Extent Tags pour Partitioning Logique

```kql
// Tag des extents par tenant pour isolation
.alter table ApplicationLogs policy extent_tags_retention 
```json
{
  "TagsToRetain": ["TenantId"]
}
```

// Requête optimisée avec tag filtering
ApplicationLogs
| where extent_tags() has "TenantId:tenant123"
| where Timestamp > ago(1d)
| summarize count() by Level
```

### 4. Row Level Security

Pour isolation multi-tenant:

```kql
.create function TenantFilteredLogs(tenantId:string) {
    ApplicationLogs
    | where TenantId == tenantId
}

// Restriction pour utilisateurs
.alter table ApplicationLogs policy restricted_view_access true

// Grant accès via fonction avec RLS
.add table ApplicationLogs viewers ('aaduser=analyst@company.com;TenantId=tenant123') 
  with (RestrictedViewAccess=TenantFilteredLogs('tenant123'))
```

## Ingestion Deep Dive

### Option 1: Event Hubs Streaming

**Configuration Event Hub**:
```bash
# Création du namespace
az eventhubs namespace create \
  --name "loganalysis-prod-eh" \
  --resource-group "rg-loganalysis-prod" \
  --location "westeurope" \
  --sku "Standard" \
  --enable-auto-inflate true \
  --maximum-throughput-units 20

# Création de l'Event Hub
az eventhubs eventhub create \
  --name "application-logs" \
  --namespace-name "loganalysis-prod-eh" \
  --partition-count 32 \
  --message-retention 1
```

**Data Connection ADX**:
```kql
.create-or-alter table ApplicationLogs ingestion eventhub data connection 'EHConnection'
```json
{
  "eventHubResourceId": "/subscriptions/.../eventhubs/application-logs",
  "consumerGroup": "adx-ingestion",
  "tableName": "ApplicationLogs",
  "mappingRuleName": "ApplicationLogsMapping",
  "dataFormat": "JSON",
  "compression": "GZip"
}
```
```

**Throughput**:
- 32 partitions × 1 MB/s = 32 MB/s = ~115 GB/h
- Avec burst: ~200-250 GB/h
- Couverture: 5 TB / 24h = 208 GB/h ✓

### Option 2: Blob Storage avec Event Grid

**Data Connection**:
```kql
.create-or-alter table ApplicationLogs ingestion blob data connection 'StorageConnection'
```json
{
  "storageAccountResourceId": "/subscriptions/.../storageAccounts/logstorageprod",
  "containerName": "application-logs",
  "eventGridResourceId": "/subscriptions/.../eventGridTopics/storage-events",
  "tableName": "ApplicationLogs",
  "mappingRuleName": "ApplicationLogsMapping",
  "dataFormat": "Parquet"
}
```
```

**Best Practices**:
- Fichiers de 100-1000 MB (optimal)
- Format Parquet pour meilleure compression
- Batch toutes les 5-10 minutes

### Monitoring de l'Ingestion

```kql
// Échecs d'ingestion
.show ingestion failures
| where FailedOn > ago(1h)
| summarize count() by FailureKind

// Statistiques d'ingestion
.show commands
| where CommandType == "DataIngestPull"
| summarize 
    IngestionCount = count(),
    TotalGB = sum(ResourceUtilization.MemoryPeak) / 1024 / 1024 / 1024
  by bin(StartedOn, 10m)
| render timechart

// Latence end-to-end
ApplicationLogs
| where Timestamp > ago(10m)
| extend IngestionLatency = ingestion_time() - Timestamp
| summarize 
    p50 = percentile(IngestionLatency, 50),
    p95 = percentile(IngestionLatency, 95),
    p99 = percentile(IngestionLatency, 99)
```

## Querying et Performance

### Optimisation des Requêtes

#### 1. Filtrage Précoce
```kql
// ❌ Mauvais: filtrage tardif
ApplicationLogs
| summarize count() by ServiceName
| where ServiceName == "AuthService"

// ✅ Bon: filtrage immédiat
ApplicationLogs
| where ServiceName == "AuthService"
| summarize count()
```

#### 2. Projection des Colonnes
```kql
// ❌ Mauvais: toutes colonnes
ApplicationLogs
| where Timestamp > ago(1d)

// ✅ Bon: colonnes nécessaires
ApplicationLogs
| where Timestamp > ago(1d)
| project Timestamp, Level, ServiceName, Message
```

#### 3. Utilisation du Cache
```kql
// Forcer l'utilisation du hot cache
ApplicationLogs
| where Timestamp > ago(7d) // Dans hot cache
| summarize count() by Level
```

### Query Patterns Avancés

#### Time Series Analysis
```kql
ApplicationLogs
| where ServiceName == "PaymentService"
| make-series 
    ErrorRate = countif(Level == "Error") * 100.0 / count()
    default=0 
    on Timestamp 
    step 5m
| extend (anomalies, score, baseline) = series_decompose_anomalies(ErrorRate, 1.5)
| where anomalies > 0
```

#### Correlation Analysis
```kql
let ErrorTraceIds = ApplicationLogs
    | where Level == "Error" and Timestamp > ago(1h)
    | distinct TraceId;
ApplicationLogs
| where TraceId in (ErrorTraceIds)
| order by TraceId, Timestamp
| project Timestamp, TraceId, ServiceName, Level, Message
```

## Sécurité et Compliance

### 1. Chiffrement
- **At Rest**: Azure Storage Encryption (AES-256) automatique
- **In Transit**: TLS 1.2+ obligatoire
- **Customer Managed Keys**: Option avec Azure Key Vault

### 2. Network Security
```bash
# VNet injection
az kusto cluster update \
  --cluster-name "loganalysis-prod-adx" \
  --resource-group "rg-loganalysis-prod" \
  --virtual-network-configuration \
    subnet-id="/subscriptions/.../subnets/adx-subnet" \
    engine-public-ip-id="/subscriptions/.../publicIPAddresses/adx-engine-ip" \
    data-management-public-ip-id="/subscriptions/.../publicIPAddresses/adx-dm-ip"
```

### 3. Data Masking
```kql
.create-or-alter function MaskPII() {
    ApplicationLogs
    | extend 
        UserId = hash_sha256(UserId),
        Email = replace_regex(Message, @"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b", "***@***.***")
}
```

## High Availability et Disaster Recovery

### 1. Cluster Redundancy
- **Minimum**: 3 nœuds (2N+1 pour quorum)
- **Availability Zones**: Distribution sur 3 zones
- **SLA**: 99.9% avec zone redundancy

### 2. Follower Clusters
```kql
// Sur cluster secondaire (read-only)
.add follower database LogAnalyticsDB from 
  cluster('https://loganalysis-prod-adx.westeurope.kusto.windows.net')
  database('LogAnalyticsDB')
```

**Use cases**:
- Read scaling (analytics workloads)
- Geo-distribution
- DR (promote follower si primaire down)

### 3. Continuous Export pour Backup
```kql
.create-or-alter continuous-export LogsBackup
over (ApplicationLogs)
to table ExportedLogs
  (Timestamp:datetime, Level:string, Message:string, [...])
with
(intervalBetweenRuns=10m, sizeLimit=1073741824)
<| 
    ApplicationLogs
    | where Timestamp > ago(1d)
```

## Best Practices

### Do's ✅
1. **Partitionner** par datetime pour queries temporelles
2. **Utiliser materialized views** pour agrégations fréquentes
3. **Compresser** les données sources (GZip, Snappy)
4. **Monitorer** l'ingestion et query performance
5. **Optimiser** le hot cache selon patterns d'accès
6. **Tagger** les extents pour multi-tenancy

### Don'ts ❌
1. **Ne pas** ingérer des fichiers < 1MB (overhead)
2. **Ne pas** faire des `select *` sans filtres
3. **Ne pas** créer trop d'indexes (automatique)
4. **Éviter** les requêtes sans time range
5. **Ne pas** stocker des PII sans masking

## Cas d'Usage Réels

### 1. Troubleshooting Production
```kql
// Find root cause d'une panne
ApplicationLogs
| where Timestamp between (datetime(2024-06-15 14:30) .. datetime(2024-06-15 15:00))
| where Level in ("Error", "Critical")
| summarize 
    ErrorCount = count(),
    AffectedUsers = dcount(UserId),
    Sample = take_any(Message, 5)
  by ServiceName, bin(Timestamp, 1m)
| order by ErrorCount desc
```

### 2. Security Audit
```kql
// Détection d'anomalies d'authentification
ApplicationLogs
| where Logger == "AuthService"
| where Message has "login"
| summarize 
    LoginAttempts = count(),
    FailedLogins = countif(ResponseCode >= 400),
    UniqueIPs = dcount(extract(@"ip=([0-9.]+)", 1, Message))
  by UserId, bin(Timestamp, 5m)
| where FailedLogins > 10 or UniqueIPs > 5
```

### 3. Business Analytics
```kql
// Analyse des patterns d'utilisation
ApplicationLogs
| where ServiceName == "ApiGateway"
| extend Endpoint = extract(@"path=([^ ]+)", 1, RequestPath)
| summarize 
    Requests = count(),
    AvgLatency = avg(DurationMs),
    P95Latency = percentile(DurationMs, 95),
    ErrorRate = countif(ResponseCode >= 500) * 100.0 / count()
  by Endpoint, bin(Timestamp, 1h)
| order by Requests desc
```

## Conclusion

Azure Data Explorer est la solution idéale pour l'analyse de 5TB/jour de logs grâce à:
- **Performance**: Sub-second queries sur billions de events
- **Scale**: Ingestion et storage quasi-illimités
- **Coût**: Compression et tiered storage optimaux
- **Flexibilité**: KQL puissant et intégrations riches

La combinaison ADX + Event Hubs + tiered storage offre le meilleur ratio performance/coût pour ce use case.
