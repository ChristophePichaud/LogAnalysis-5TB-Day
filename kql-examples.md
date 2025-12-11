# Exemples KQL pour Analyse de Logs

## Introduction à KQL (Kusto Query Language)

KQL est un langage de requête optimisé pour l'analyse de grands volumes de données, particulièrement efficace pour les logs et télémétries. Ce document présente des exemples pratiques pour l'analyse quotidienne de logs.

## Structure de Base d'une Requête KQL

```kql
TableName
| where Condition
| extend NewColumn = Expression
| summarize Aggregation by GroupBy
| order by Column
| take N
```

## Exemples par Cas d'Usage

### 1. Analyses de Base

#### Compter les logs par niveau de sévérité
```kql
ApplicationLogs
| where Timestamp > ago(24h)
| summarize Count = count() by Level
| order by Count desc
| render piechart
```

#### Top 10 services les plus verbeux
```kql
ApplicationLogs
| where Timestamp > ago(1h)
| summarize LogCount = count() by ServiceName
| top 10 by LogCount desc
```

#### Logs d'erreur avec contexte
```kql
ApplicationLogs
| where Level in ("Error", "Critical")
| where Timestamp > ago(1h)
| project Timestamp, ServiceName, Message, Exception, TraceId
| order by Timestamp desc
| take 100
```

### 2. Analyse Temporelle

#### Distribution horaire des logs
```kql
ApplicationLogs
| where Timestamp > ago(7d)
| summarize Count = count() by bin(Timestamp, 1h), Level
| render timechart
```

#### Comparaison jour vs jour
```kql
ApplicationLogs
| where Timestamp > ago(48h)
| extend DayCategory = iff(Timestamp > ago(24h), "Today", "Yesterday")
| summarize Count = count() by bin(Timestamp, 1h), DayCategory
| render timechart
```

#### Pattern hebdomadaire
```kql
ApplicationLogs
| where Timestamp > ago(30d)
| extend DayOfWeek = dayofweek(Timestamp)
| extend Hour = hourofday(Timestamp)
| summarize AvgLogs = avg(todouble(1)) by DayOfWeek, Hour
| render heatmap
```

### 3. Performance et Latence

#### Analyse de latence par endpoint
```kql
ApplicationLogs
| where ServiceName == "ApiGateway"
| where Timestamp > ago(1h)
| extend Endpoint = extract(@"endpoint=([^ ]+)", 1, Message)
| summarize 
    RequestCount = count(),
    P50 = percentile(DurationMs, 50),
    P95 = percentile(DurationMs, 95),
    P99 = percentile(DurationMs, 99),
    Max = max(DurationMs)
  by Endpoint
| where RequestCount > 100
| order by P95 desc
```

#### Requêtes lentes (SLO > 3 secondes)
```kql
ApplicationLogs
| where DurationMs > 3000
| where Timestamp > ago(24h)
| extend Endpoint = extract(@"path=([^ ]+)", 1, RequestPath)
| summarize 
    SlowRequests = count(),
    AvgDuration = avg(DurationMs),
    MaxDuration = max(DurationMs)
  by Endpoint, ServiceName
| order by SlowRequests desc
```

#### Time series de latence avec détection d'anomalies
```kql
ApplicationLogs
| where ServiceName == "PaymentService"
| where Timestamp > ago(7d)
| summarize AvgDuration = avg(DurationMs) by bin(Timestamp, 5m)
| extend (anomalies, score, baseline) = series_decompose_anomalies(AvgDuration, 1.5)
| render anomalychart with (anomalycolumns=anomalies)
```

### 4. Analyse d'Erreurs

#### Top erreurs par fréquence
```kql
ApplicationLogs
| where Level == "Error"
| where Timestamp > ago(24h)
| extend ErrorType = extract(@"Exception: ([^:]+)", 1, Exception)
| summarize 
    ErrorCount = count(),
    AffectedUsers = dcount(UserId),
    AffectedServices = dcount(ServiceName),
    FirstOccurrence = min(Timestamp),
    LastOccurrence = max(Timestamp),
    SampleMessage = any(Message)
  by ErrorType
| order by ErrorCount desc
| take 20
```

#### Cascade d'erreurs (erreurs liées par TraceId)
```kql
let ErrorTraces = ApplicationLogs
    | where Level == "Error" and Timestamp > ago(1h)
    | where ServiceName == "OrderService"
    | distinct TraceId;
ApplicationLogs
| where TraceId in (ErrorTraces)
| order by TraceId, Timestamp asc
| project Timestamp, TraceId, ServiceName, Level, Message, DurationMs
```

#### Taux d'erreur par service
```kql
ApplicationLogs
| where Timestamp > ago(24h)
| summarize 
    TotalRequests = count(),
    ErrorCount = countif(Level in ("Error", "Critical")),
    ErrorRate = round(countif(Level in ("Error", "Critical")) * 100.0 / count(), 2)
  by ServiceName, bin(Timestamp, 1h)
| where TotalRequests > 100
| order by ErrorRate desc
| render timechart
```

### 5. Sécurité et Audit

#### Tentatives de connexion échouées
```kql
ApplicationLogs
| where Logger contains "Auth"
| where Message has_any ("login failed", "authentication failed")
| where Timestamp > ago(1h)
| extend IPAddress = extract(@"ip=([0-9.]+)", 1, Message)
| summarize 
    FailedAttempts = count(),
    UniqueUsers = dcount(UserId),
    FirstAttempt = min(Timestamp),
    LastAttempt = max(Timestamp)
  by IPAddress
| where FailedAttempts > 5
| order by FailedAttempts desc
```

#### Détection d'attaques par force brute
```kql
ApplicationLogs
| where Timestamp > ago(5m)
| where ResponseCode in (401, 403)
| extend IPAddress = extract(@"ip=([0-9.]+)", 1, Message)
| summarize 
    Attempts = count(),
    TargetedAccounts = make_set(UserId, 10)
  by IPAddress, bin(Timestamp, 1m)
| where Attempts > 10
| project Timestamp, IPAddress, Attempts, TargetedAccounts
```

#### Accès à ressources sensibles
```kql
ApplicationLogs
| where Timestamp > ago(7d)
| where RequestPath has_any ("/admin", "/api/sensitive", "/internal")
| summarize 
    AccessCount = count(),
    UniqueUsers = dcount(UserId),
    SuccessfulAccess = countif(ResponseCode < 400),
    BlockedAccess = countif(ResponseCode >= 400)
  by UserId, RequestPath
| where AccessCount > 0
| order by AccessCount desc
```

### 6. Analyse Multi-Service (Distributed Tracing)

#### Vue complète d'une transaction
```kql
let traceId = "abc123-def456-ghi789";
ApplicationLogs
| where TraceId == traceId
| order by Timestamp asc
| project 
    Timestamp,
    ServiceName,
    Level,
    Message,
    DurationMs,
    ResponseCode,
    Step = row_number()
| extend TotalDuration = toscalar(
    ApplicationLogs 
    | where TraceId == traceId 
    | summarize max(Timestamp) - min(Timestamp)
  ) / 1ms
```

#### Services les plus lents dans la chaîne
```kql
ApplicationLogs
| where Timestamp > ago(1h)
| where isnotempty(TraceId)
| summarize ServiceDuration = sum(DurationMs) by TraceId, ServiceName
| summarize 
    AvgDuration = avg(ServiceDuration),
    P95Duration = percentile(ServiceDuration, 95),
    CallCount = count()
  by ServiceName
| order by P95Duration desc
```

#### Traçage des dépendances entre services
```kql
ApplicationLogs
| where Timestamp > ago(1h)
| where isnotempty(TraceId)
| summarize by TraceId, ServiceName
| join kind=inner (
    ApplicationLogs
    | where Timestamp > ago(1h)
    | where isnotempty(TraceId)
    | summarize by TraceId, ServiceName
  ) on TraceId
| where ServiceName != ServiceName1
| summarize Count = count() by Caller = ServiceName, Callee = ServiceName1
| where Count > 10
| render force_directed_graph 
```

### 7. Analyse Business

#### Utilisateurs les plus actifs
```kql
ApplicationLogs
| where Timestamp > ago(24h)
| where isnotempty(UserId)
| summarize 
    Actions = count(),
    UniqueEndpoints = dcount(RequestPath),
    TotalDataTransferred = sum(BytesSent + BytesReceived),
    AvgResponseTime = avg(DurationMs),
    ErrorCount = countif(ResponseCode >= 400)
  by UserId
| order by Actions desc
| take 50
```

#### Analyse géographique du trafic
```kql
ApplicationLogs
| where Timestamp > ago(7d)
| extend IPAddress = extract(@"ip=([0-9.]+)", 1, Message)
| extend GeoInfo = geo_info_from_ip_address(IPAddress)
| extend Country = tostring(GeoInfo.country), City = tostring(GeoInfo.city)
| where isnotempty(Country)
| summarize 
    Requests = count(),
    UniqueUsers = dcount(UserId),
    AvgLatency = avg(DurationMs)
  by Country, City
| order by Requests desc
| take 20
```

#### Funnel d'utilisation
```kql
let StartEvent = "page_view:home";
let Step1 = "page_view:product";
let Step2 = "action:add_to_cart";
let Step3 = "action:checkout";
let users_started = ApplicationLogs
    | where Timestamp > ago(7d)
    | where Message has StartEvent
    | distinct UserId;
let users_step1 = ApplicationLogs
    | where Timestamp > ago(7d)
    | where Message has Step1
    | distinct UserId;
let users_step2 = ApplicationLogs
    | where Timestamp > ago(7d)
    | where Message has Step2
    | distinct UserId;
let users_step3 = ApplicationLogs
    | where Timestamp > ago(7d)
    | where Message has Step3
    | distinct UserId;
print 
    Started = toscalar(users_started | count),
    ViewedProduct = toscalar(users_step1 | count),
    AddedToCart = toscalar(users_step2 | count),
    Checkout = toscalar(users_step3 | count)
```

### 8. Monitoring et Alerting

#### SLA monitoring (availability)
```kql
ApplicationLogs
| where Timestamp > ago(30d)
| where ServiceName == "ApiGateway"
| summarize 
    TotalRequests = count(),
    SuccessfulRequests = countif(ResponseCode < 500),
    Availability = round(countif(ResponseCode < 500) * 100.0 / count(), 3)
  by bin(Timestamp, 1d)
| extend SLAMet = Availability >= 99.9
| project Timestamp, Availability, SLAMet
```

#### Détection de pics anormaux
```kql
ApplicationLogs
| where Timestamp > ago(30d)
| summarize RequestCount = count() by bin(Timestamp, 5m)
| extend 
    Baseline = series_stats_dynamic(RequestCount).avg,
    StdDev = series_stats_dynamic(RequestCount).stdev
| extend Threshold = Baseline + (3 * StdDev)
| where RequestCount > Threshold
| project Timestamp, RequestCount, Baseline, Threshold
```

#### Health check des services
```kql
ApplicationLogs
| where Timestamp > ago(5m)
| summarize LastSeen = max(Timestamp) by ServiceName
| extend 
    Status = case(
        LastSeen > ago(2m), "Healthy",
        LastSeen > ago(5m), "Warning",
        "Critical"
    ),
    MinutesSinceLastLog = datetime_diff('minute', now(), LastSeen)
| project ServiceName, Status, MinutesSinceLastLog, LastSeen
| order by Status desc, MinutesSinceLastLog desc
```

### 9. Optimisation des Coûts

#### Top services par volume de logs
```kql
ApplicationLogs
| where Timestamp > ago(30d)
| summarize 
    LogCount = count(),
    EstimatedSizeGB = (count() * 500.0) / 1024 / 1024 / 1024 // Assuming ~500 bytes/log
  by ServiceName
| order by EstimatedSizeGB desc
| extend CumulativePercentage = round(100.0 * row_cumsum(EstimatedSizeGB) / toscalar(summarize sum(EstimatedSizeGB)), 2)
```

#### Logs de faible valeur (candidats pour rétention réduite)
```kql
ApplicationLogs
| where Level == "Debug" or Level == "Trace"
| where Timestamp > ago(7d)
| summarize 
    Count = count(),
    SizeEstimateGB = (count() * 500.0) / 1024 / 1024 / 1024
  by ServiceName, Level
| order by SizeEstimateGB desc
```

### 10. Requêtes Avancées

#### Analyse de patterns avec regex
```kql
ApplicationLogs
| where Timestamp > ago(24h)
| extend 
    OrderId = extract(@"orderId=([A-Z0-9-]+)", 1, Message),
    Amount = todouble(extract(@"amount=(\d+\.?\d*)", 1, Message))
| where isnotempty(OrderId)
| summarize 
    TotalAmount = sum(Amount),
    OrderCount = dcount(OrderId)
  by ServiceName
```

#### Join avec table de référence
```kql
// Table de référence des services
let ServiceMetadata = datatable(ServiceName:string, Team:string, Tier:string)
[
    "AuthService", "Platform", "Critical",
    "PaymentService", "Finance", "Critical",
    "NotificationService", "Communication", "Standard"
];
ApplicationLogs
| where Timestamp > ago(1h)
| summarize ErrorCount = countif(Level == "Error") by ServiceName
| join kind=leftouter ServiceMetadata on ServiceName
| where Tier == "Critical"
| order by ErrorCount desc
```

#### Pivot dynamique
```kql
ApplicationLogs
| where Timestamp > ago(24h)
| summarize Count = count() by Level, ServiceName
| evaluate pivot(Level, sum(Count))
```

#### Analyse de séries temporelles avec prédiction
```kql
ApplicationLogs
| where Timestamp > ago(30d)
| where ServiceName == "ApiGateway"
| summarize RequestCount = count() by bin(Timestamp, 1h)
| extend (predictions, pred_score) = series_decompose_forecast(RequestCount, 24) // Prédire 24h
| render timechart
```

## Fonctions Personnalisées

### Fonction pour standardiser les logs
```kql
.create-or-alter function ParseApplicationLog(logLevel:string) {
    ApplicationLogs
    | where Level == logLevel
    | extend 
        NormalizedTimestamp = bin(Timestamp, 1m),
        ErrorCategory = case(
            Exception contains "Timeout", "Timeout",
            Exception contains "Connection", "Connection",
            Exception contains "Authentication", "Auth",
            "Other"
        )
    | project 
        NormalizedTimestamp,
        ServiceName,
        ErrorCategory,
        Message,
        TraceId
}

// Utilisation
ParseApplicationLog("Error")
| where NormalizedTimestamp > ago(1h)
| summarize count() by ErrorCategory
```

### Fonction pour calcul de SLI
```kql
.create-or-alter function CalculateSLI(serviceName:string, windowSize:timespan) {
    ApplicationLogs
    | where ServiceName == serviceName
    | where Timestamp > ago(windowSize)
    | summarize 
        TotalRequests = count(),
        GoodRequests = countif(ResponseCode < 500 and DurationMs < 1000),
        ErrorRequests = countif(ResponseCode >= 500),
        SlowRequests = countif(DurationMs >= 1000)
    | extend 
        SLI = round(todouble(GoodRequests) * 100.0 / TotalRequests, 2),
        ErrorRate = round(todouble(ErrorRequests) * 100.0 / TotalRequests, 2),
        SlowRate = round(todouble(SlowRequests) * 100.0 / TotalRequests, 2)
    | project SLI, ErrorRate, SlowRate, TotalRequests
}

// Utilisation
CalculateSLI("PaymentService", 1h)
```

## Best Practices KQL

### Performance
1. **Filtrer tôt**: Toujours utiliser `where` avant `summarize` ou `join`
2. **Limiter les colonnes**: Utiliser `project` pour réduire les données
3. **Utiliser le cache**: Requêtes sur hot cache (< 14 jours par défaut)
4. **Éviter les scans complets**: Toujours spécifier une plage temporelle

### Lisibilité
1. **Indentation**: Une opération par ligne
2. **Variables**: Utiliser `let` pour requêtes complexes
3. **Commentaires**: Documenter la logique métier
4. **Naming**: Noms de colonnes explicites

### Exemple de requête bien structurée
```kql
// Analyse des erreurs critiques avec impact business
// Author: Platform Team | Date: 2024-06-15
let timeRange = 24h;
let errorThreshold = 100;
let criticalServices = dynamic(["PaymentService", "AuthService", "OrderService"]);
//
ApplicationLogs
| where Timestamp > ago(timeRange)                    // Filtrage temporel
| where Level in ("Error", "Critical")                // Filtrage par sévérité
| where ServiceName in (criticalServices)             // Services critiques uniquement
| extend ErrorCategory = extract(@"^(\w+)Exception", 1, Exception)
| summarize 
    ErrorCount = count(),
    AffectedUsers = dcount(UserId),
    UniqueErrors = dcount(ErrorCategory),
    SampleMessages = make_set(Message, 3)
  by ServiceName, ErrorCategory
| where ErrorCount > errorThreshold                   // Seuil significatif
| order by ErrorCount desc
| project 
    ServiceName,
    ErrorCategory,
    ErrorCount,
    AffectedUsers,
    SampleMessages
```

## Ressources Complémentaires

- [KQL Reference officielle](https://docs.microsoft.com/azure/data-explorer/kusto/query/)
- [KQL Cheat Sheet](https://github.com/marcusbakker/KQL)
- [Best Practices KQL](https://docs.microsoft.com/azure/data-explorer/kusto/query/best-practices)
- [Tutoriels interactifs](https://dataexplorer.azure.com/clusters/help/databases/Samples)

## Playground pour Pratique

Microsoft propose un cluster de démonstration avec données samples:
- URL: https://dataexplorer.azure.com/
- Cluster: help
- Database: Samples
- Tables: StormEvents, ContosoSales, etc.

Testez vos requêtes avant de les déployer en production!
