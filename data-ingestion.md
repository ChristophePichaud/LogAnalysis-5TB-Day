# Stratégie d'Ingestion de Données

## Vue d'ensemble

L'ingestion de 5TB de logs par jour (~208 GB/heure en moyenne) nécessite une architecture robuste, scalable et résiliente. Ce document détaille les différentes stratégies d'ingestion pour Azure Data Explorer.

## Volume et Contraintes

### Métriques Cibles
- **Volume quotidien**: 5 TB (non compressé)
- **Débit moyen**: ~208 GB/heure (~58 MB/seconde)
- **Débit peak**: ~625 GB/heure (facteur 3x aux heures de pointe)
- **Latence souhaitée**: < 2 minutes end-to-end
- **Format principal**: JSON structuré
- **Compression ratio**: 10:1 (5TB → 500GB stocké)

### Contraintes Techniques
- **Limites ADX**: 
  - Ingestion: ~200 GB/heure/nœud
  - Fichiers optimaux: 100MB - 1GB
  - Batch size: 1GB ou 1000 blobs
- **Network**: Bandwidth suffisant (10+ Gbps)
- **Event Hub**: 32+ partitions recommandées

## Options d'Ingestion

### Option 1: Azure Event Hubs (Streaming) - ⭐ RECOMMANDÉ

#### Architecture
```
Applications/Services
    ↓
Agents de collecte (Fluentd/Logstash/Vector)
    ↓
Azure Event Hubs (32+ partitions)
    ↓
ADX Data Connection (Native integration)
    ↓
Azure Data Explorer Tables
```

#### Avantages
- ✅ **Faible latence**: < 30 secondes à 2 minutes
- ✅ **Découplage**: Buffer entre producers et consumers
- ✅ **Résilience**: Retry automatique, dead-letter queue
- ✅ **Scaling**: Auto-inflate jusqu'à 40 TUs
- ✅ **Simplicité**: Gestion automatique par ADX

#### Configuration Event Hub

**Namespace**:
```bash
az eventhubs namespace create \
  --name "loganalysis-prod-eh" \
  --resource-group "rg-loganalysis-prod" \
  --location "westeurope" \
  --sku "Standard" \
  --enable-auto-inflate true \
  --maximum-throughput-units 20 \
  --capacity 10
```

**Event Hub**:
```bash
az eventhubs eventhub create \
  --name "application-logs" \
  --namespace-name "loganalysis-prod-eh" \
  --partition-count 32 \
  --message-retention 1 \
  --cleanup-policy "Delete"
```

**Paramètres clés**:
- **Partitions**: 32 (permet 32 MB/s = ~115 GB/h)
- **Throughput Units**: 10-20 (1 TU = 1 MB/s ingress)
- **Retention**: 1 jour (suffisant pour buffer)
- **Auto-inflate**: Activé pour peaks

#### Calcul du dimensionnement

```
Débit peak: 625 GB/h = 173 MB/s
Avec compression 2:1 (avant Event Hub): ~86 MB/s
Throughput Units requis: 86 TU
Avec auto-inflate: Commencer à 10 TU, scaler à 20 TU

Partitions pour parallélisme ADX:
32 partitions × 1 MB/s = 32 MB/s base
Avec burst: ~100-150 MB/s
Couverture: 86 MB/s ✓
```

#### Data Connection ADX

```kql
.create-or-alter table ApplicationLogs ingestion eventhub data connection 'ApplicationLogsConnection'
```json
{
  "properties": {
    "eventHubResourceId": "/subscriptions/{sub-id}/resourceGroups/rg-loganalysis-prod/providers/Microsoft.EventHub/namespaces/loganalysis-prod-eh/eventhubs/application-logs",
    "consumerGroup": "adx-consumer",
    "tableName": "ApplicationLogs",
    "mappingRuleName": "ApplicationLogsMapping",
    "dataFormat": "JSON",
    "compression": "GZip",
    "eventSystemProperties": ["x-opt-enqueued-time"],
    "managedIdentityResourceId": "/subscriptions/{sub-id}/resourceGroups/rg-loganalysis-prod/providers/Microsoft.ManagedIdentity/userAssignedIdentities/adx-identity"
  }
}
```
```

#### Configuration des Agents de Collecte

**Fluentd**:
```yaml
<source>
  @type tail
  path /var/log/application/*.log
  pos_file /var/log/td-agent/application.log.pos
  tag application.logs
  format json
  time_key timestamp
  time_format %Y-%m-%dT%H:%M:%S.%LZ
</source>

<filter application.logs>
  @type record_transformer
  enable_ruby true
  <record>
    hostname "#{Socket.gethostname}"
    environment "#{ENV['ENVIRONMENT']}"
    region "#{ENV['AZURE_REGION']}"
  </record>
</filter>

<match application.logs>
  @type azure_event_hubs
  connection_string "#{ENV['EVENT_HUB_CONNECTION_STRING']}"
  hub_name "application-logs"
  
  # Batching pour efficacité
  <buffer>
    @type file
    path /var/log/td-agent/buffer/eventhub
    flush_mode interval
    flush_interval 10s
    chunk_limit_size 5MB
    total_limit_size 1GB
    retry_type exponential_backoff
    retry_max_interval 30s
  </buffer>
  
  # Compression
  compress gzip
</match>
```

**Vector** (Alternative moderne):
```toml
[sources.application_logs]
type = "file"
include = ["/var/log/application/*.log"]
data_dir = "/var/lib/vector"

[transforms.parse_json]
type = "remap"
inputs = ["application_logs"]
source = '''
  . = parse_json!(.message)
  .hostname = get_hostname!()
  .environment = get_env_var!("ENVIRONMENT")
'''

[sinks.azure_event_hub]
type = "azure_event_hubs"
inputs = ["parse_json"]
connection_string = "${EVENT_HUB_CONNECTION_STRING}"
hub_name = "application-logs"
encoding.codec = "json"

# Batching
batch.max_bytes = 5242880  # 5MB
batch.timeout_secs = 10

# Compression
compression = "gzip"
```

#### Monitoring Event Hub

```kql
// Métriques Event Hub dans ADX
.show commands
| where CommandType == "DataIngestPull"
| where OriginalClientRequestId contains "eventhub"
| summarize 
    EventCount = sum(RowCount),
    TotalSizeGB = sum(ResourceUtilization.MemoryPeak) / 1024 / 1024 / 1024,
    AvgLatencySeconds = avg(Duration) / 1000.0
  by bin(StartedOn, 5m)
| render timechart
```

---

### Option 2: Azure Blob Storage (Batch) - Pour Historique

#### Architecture
```
Applications/Services
    ↓
Agents de collecte
    ↓
Azure Blob Storage (hourly batches)
    ↓
Event Grid Notification
    ↓
ADX Data Connection
    ↓
Azure Data Explorer Tables
```

#### Avantages
- ✅ **Coût réduit**: Pas de Event Hub
- ✅ **Simplicité**: Drop de fichiers
- ✅ **Réingestion**: Facile à rejouer
- ✅ **Formats variés**: JSON, Parquet, CSV, Avro

#### Inconvénients
- ⚠️ **Latence plus élevée**: 5-15 minutes
- ⚠️ **Moins de résilience**: Pas de buffer

#### Configuration Storage Account

```bash
az storage account create \
  --name "loganalyticsprodsa" \
  --resource-group "rg-loganalysis-prod" \
  --location "westeurope" \
  --sku "Standard_LRS" \
  --kind "StorageV2" \
  --access-tier "Hot" \
  --enable-hierarchical-namespace true  # Data Lake Gen2

az storage container create \
  --account-name "loganalyticsprodsa" \
  --name "application-logs" \
  --public-access off
```

#### Event Grid Configuration

```bash
# Event Grid Topic
az eventgrid topic create \
  --name "storage-events-topic" \
  --resource-group "rg-loganalysis-prod" \
  --location "westeurope"

# Event Grid Subscription
az eventgrid event-subscription create \
  --name "blob-created-subscription" \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/rg-loganalysis-prod/providers/Microsoft.Storage/storageAccounts/loganalyticsprodsa" \
  --endpoint-type "azurefunction" \
  --endpoint "/subscriptions/{sub-id}/resourceGroups/rg-loganalysis-prod/providers/Microsoft.Web/sites/adx-ingestion-func/functions/BlobCreatedHandler" \
  --included-event-types "Microsoft.Storage.BlobCreated"
```

#### Data Connection ADX

```kql
.create-or-alter table ApplicationLogs ingestion blob data connection 'StorageConnection'
```json
{
  "properties": {
    "storageAccountResourceId": "/subscriptions/{sub-id}/resourceGroups/rg-loganalysis-prod/providers/Microsoft.Storage/storageAccounts/loganalyticsprodsa",
    "containerName": "application-logs",
    "eventGridResourceId": "/subscriptions/{sub-id}/resourceGroups/rg-loganalysis-prod/providers/Microsoft.EventGrid/topics/storage-events-topic",
    "tableName": "ApplicationLogs",
    "mappingRuleName": "ApplicationLogsMapping",
    "dataFormat": "Parquet",
    "ignoreFirstRecord": false,
    "managedIdentityResourceId": "/subscriptions/{sub-id}/resourceGroups/rg-loganalysis-prod/providers/Microsoft.ManagedIdentity/userAssignedIdentities/adx-identity"
  }
}
```
```

#### Pattern de Naming des Fichiers

```
application-logs/
  └── year=2024/
      └── month=06/
          └── day=15/
              └── hour=14/
                  ├── app-logs-20240615-140000-001.parquet.gz
                  ├── app-logs-20240615-140000-002.parquet.gz
                  └── app-logs-20240615-140000-003.parquet.gz
```

**Best practices**:
- **Taille**: 100-500 MB par fichier (après compression)
- **Format**: Parquet (meilleure compression et performance)
- **Compression**: GZip ou Snappy
- **Fréquence**: Toutes les 5-10 minutes

#### Script de Upload

```python
from azure.storage.blob import BlobServiceClient
import gzip
import json
from datetime import datetime
import pyarrow as pa
import pyarrow.parquet as pq

class LogUploader:
    def __init__(self, connection_string, container_name):
        self.blob_service = BlobServiceClient.from_connection_string(connection_string)
        self.container_client = self.blob_service.get_container_client(container_name)
    
    def upload_logs_parquet(self, logs, partition_date):
        # Conversion en Parquet
        table = pa.Table.from_pylist(logs)
        
        # Path avec partitioning
        year = partition_date.strftime("%Y")
        month = partition_date.strftime("%m")
        day = partition_date.strftime("%d")
        hour = partition_date.strftime("%H")
        timestamp = partition_date.strftime("%Y%m%d-%H%M%S")
        
        blob_path = f"year={year}/month={month}/day={day}/hour={hour}/logs-{timestamp}.parquet.gz"
        
        # Write to buffer
        import io
        buffer = io.BytesIO()
        pq.write_table(table, buffer, compression='gzip')
        buffer.seek(0)
        
        # Upload
        blob_client = self.container_client.get_blob_client(blob_path)
        blob_client.upload_blob(buffer, overwrite=False)
        
        print(f"Uploaded {len(logs)} logs to {blob_path}")

# Usage
uploader = LogUploader(connection_string, "application-logs")
uploader.upload_logs_parquet(log_batch, datetime.now())
```

---

### Option 3: Hybrid (Event Hub + Storage)

#### Use Cases
- **Event Hub**: Logs critiques temps réel (erreurs, sécurité)
- **Storage**: Logs bulk non-critiques (debug, trace)

#### Routage par niveau de sévérité

```python
def route_log(log_entry):
    level = log_entry.get('level', 'info').lower()
    
    if level in ['error', 'critical', 'warning']:
        # Route vers Event Hub pour traitement immédiat
        send_to_event_hub(log_entry)
    else:
        # Route vers Storage pour batch processing
        send_to_storage(log_entry)
```

#### Configuration Fluentd avec Routing

```yaml
<match application.logs>
  @type copy
  
  # Logs critiques vers Event Hub
  <store>
    @type azure_event_hubs
    connection_string "#{ENV['EVENT_HUB_CONNECTION_STRING']}"
    hub_name "critical-logs"
    <filter>
      @type grep
      <regexp>
        key level
        pattern /(error|critical|warning)/i
      </regexp>
    </filter>
  </store>
  
  # Tous les logs vers Storage (batch)
  <store>
    @type azure_storage_append_blob
    azure_storage_account "#{ENV['STORAGE_ACCOUNT']}"
    azure_storage_access_key "#{ENV['STORAGE_KEY']}"
    azure_container "application-logs"
    path "logs/%Y/%m/%d/%H/logs-#{Socket.gethostname}-%M.json"
    
    <buffer time>
      @type file
      path /var/log/td-agent/buffer/storage
      timekey 600  # 10 minutes
      timekey_wait 60
      chunk_limit_size 256MB
    </buffer>
  </store>
</match>
```

---

## Formats de Données

### JSON (Recommandé pour démarrage)

**Avantages**:
- Flexible et facile à débugger
- Support natif de dynamic types dans ADX
- Lisible par humains

**Inconvénients**:
- Taille plus grande (~30-40% vs Parquet)
- Parsing moins performant

**Exemple**:
```json
{
  "timestamp": "2024-06-15T14:35:42.123Z",
  "level": "error",
  "logger": "com.company.OrderService",
  "message": "Failed to process order",
  "exception": "OrderProcessingException: Timeout while contacting payment service",
  "traceId": "abc123-def456-ghi789",
  "spanId": "span-001",
  "serviceName": "OrderService",
  "environment": "production",
  "properties": {
    "orderId": "ORD-123456",
    "userId": "user-789",
    "amount": 99.99,
    "retryCount": 3
  }
}
```

### Parquet (Recommandé pour production)

**Avantages**:
- Compression excellente (50-70% vs JSON)
- Lecture columnaire très rapide
- Schéma fort et typé

**Inconvénients**:
- Nécessite conversion côté agent
- Moins flexible pour schémas changeants

**Configuration**:
```python
import pyarrow as pa
import pyarrow.parquet as pq

# Définition du schéma
schema = pa.schema([
    ('timestamp', pa.timestamp('ms')),
    ('level', pa.string()),
    ('logger', pa.string()),
    ('message', pa.string()),
    ('exception', pa.string()),
    ('traceId', pa.string()),
    ('serviceName', pa.string()),
    ('properties', pa.string())  # JSON stringifié
])

# Écriture
pq.write_table(
    table,
    'logs.parquet',
    compression='snappy',
    use_dictionary=True
)
```

### CSV (Pour simplicité)

**Use case**: Logs simples, outils legacy

**Format**:
```csv
timestamp,level,serviceName,message
2024-06-15T14:35:42Z,error,OrderService,"Failed to process order"
```

---

## Transformation et Enrichissement

### Update Policy pour Enrichissement

```kql
// Fonction d'enrichissement
.create-or-alter function EnrichLogs() {
    ApplicationLogs
    | extend 
        // Parsing enrichi
        Severity = case(
            Level == "critical", 5,
            Level == "error", 4,
            Level == "warning", 3,
            Level == "info", 2,
            1
        ),
        // Extraction de métadonnées
        ErrorCode = extract(@"errorCode=(\d+)", 1, Message),
        IPAddress = extract(@"ip=([0-9.]+)", 1, Message),
        // Géolocalisation
        GeoInfo = geo_info_from_ip_address(extract(@"ip=([0-9.]+)", 1, Message)),
        // Catégorisation
        ServiceCategory = extract(@"^([^-]+)", 1, ServiceName),
        // Date parts pour partitioning
        Year = getyear(Timestamp),
        Month = getmonth(Timestamp),
        Day = dayofmonth(Timestamp),
        Hour = hourofday(Timestamp)
    | extend
        Country = tostring(GeoInfo.country),
        City = tostring(GeoInfo.city)
    | project-away GeoInfo
}

// Table enrichie
.create table EnrichedApplicationLogs (
    Timestamp: datetime,
    Level: string,
    Severity: int,
    ServiceName: string,
    ServiceCategory: string,
    Message: string,
    ErrorCode: string,
    IPAddress: string,
    Country: string,
    City: string,
    TraceId: string,
    Year: int,
    Month: int,
    Day: int,
    Hour: int
)

// Update policy
.alter table EnrichedApplicationLogs policy update 
@'[{
    "Source": "ApplicationLogs",
    "Query": "EnrichLogs()",
    "IsEnabled": true,
    "IsTransactional": true
}]'
```

---

## Monitoring et Troubleshooting

### Dashboard d'Ingestion

```kql
// Vue d'ensemble de l'ingestion
.show commands
| where CommandType in ("DataIngestPull", "DataIngestPush")
| where StartedOn > ago(1h)
| summarize 
    IngestedRows = sum(RowCount),
    IngestedGB = sum(ResourceUtilization.MemoryPeak) / 1024 / 1024 / 1024,
    AvgLatencySeconds = avg(Duration) / 1000.0,
    P95LatencySeconds = percentile(Duration, 95) / 1000.0,
    FailedCount = countif(State == "Failed")
  by bin(StartedOn, 5m)
| render timechart
```

### Détection d'Échecs

```kql
// Échecs d'ingestion récents
.show ingestion failures
| where FailedOn > ago(1h)
| project 
    FailedOn,
    Database,
    Table,
    OperationId,
    ErrorCode,
    FailureStatus,
    Details
| order by FailedOn desc
```

### Alertes Recommandées

**Azure Monitor Alerts**:

1. **Ingestion Rate Drop**
```kql
// Alert si ingestion < 50% de la moyenne
let baseline = ApplicationLogs
    | where ingestion_time() > ago(7d)
    | summarize AvgRowsPerMinute = count() / (7 * 24 * 60);
ApplicationLogs
| where ingestion_time() > ago(5m)
| summarize CurrentRowsPerMinute = count() / 5
| extend Baseline = toscalar(baseline)
| where CurrentRowsPerMinute < (Baseline * 0.5)
```

2. **Ingestion Latency High**
```kql
// Alert si latence > 5 minutes
ApplicationLogs
| where ingestion_time() > ago(10m)
| extend IngestionLatency = ingestion_time() - Timestamp
| summarize P95Latency = percentile(IngestionLatency, 95)
| where P95Latency > 5m
```

3. **Failed Ingestions**
```kql
// Alert si échecs > 1% du volume
.show ingestion failures
| where FailedOn > ago(5m)
| summarize FailureCount = count()
| extend Threshold = 100  // Ajuster selon volume
| where FailureCount > Threshold
```

---

## Best Practices

### Sizing et Performance

1. **Taille des fichiers**: 100MB - 1GB (optimal)
2. **Batch frequency**: 5-10 minutes
3. **Compression**: Toujours activer (GZip ou Snappy)
4. **Partitions Event Hub**: Au moins 1 partition par 1 MB/s

### Résilience

1. **Retry Logic**: Exponentiel backoff
2. **Dead Letter Queue**: Pour messages non ingérables
3. **Monitoring continu**: Alertes sur échecs/latence
4. **Backfill capability**: Capacité de réingérer depuis Storage

### Sécurité

1. **Managed Identities**: Pas de secrets dans config
2. **Private Endpoints**: Connexions privées
3. **Encryption**: At rest et in transit
4. **Audit Logging**: Tracer toutes opérations d'ingestion

### Coûts

1. **Compression**: Réduire network et storage costs
2. **Batch vs Streaming**: Storage moins cher que Event Hub
3. **Reserved Capacity**: Pour Event Hub et ADX
4. **Lifecycle policies**: Archiver logs anciens

---

## Checklist de Déploiement

### Phase 1: Setup Infrastructure
- [ ] Créer cluster ADX
- [ ] Créer Event Hub / Storage Account
- [ ] Configurer Event Grid (si Storage)
- [ ] Setup Managed Identities
- [ ] Configurer Private Endpoints

### Phase 2: Configuration ADX
- [ ] Créer Database et Tables
- [ ] Définir Ingestion Mappings
- [ ] Configurer Data Connections
- [ ] Setup Update Policies
- [ ] Configurer Retention Policies

### Phase 3: Agents et Collection
- [ ] Déployer agents de collecte
- [ ] Configurer routing et buffering
- [ ] Tester ingestion end-to-end
- [ ] Valider formats et mappings

### Phase 4: Monitoring
- [ ] Configurer dashboards d'ingestion
- [ ] Setup alertes critiques
- [ ] Documenter runbooks
- [ ] Tester procédures d'incident

### Phase 5: Optimization
- [ ] Tuner batch sizes
- [ ] Optimiser compression
- [ ] Ajuster capacités
- [ ] Réviser coûts

---

## Ressources

- [ADX Data Ingestion Overview](https://docs.microsoft.com/azure/data-explorer/ingest-data-overview)
- [Event Hub Best Practices](https://docs.microsoft.com/azure/event-hubs/event-hubs-messaging-exceptions)
- [Fluentd Documentation](https://docs.fluentd.org/)
- [Vector Documentation](https://vector.dev/docs/)
