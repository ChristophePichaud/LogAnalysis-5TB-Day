
# Step 12 : Schémas d'architecture cible (PlantUML)

## Objectif
Fournir des schémas d'architecture de synthèse (PlantUML ou autre) pour illustrer la solution cible.

## 1. Schéma global de la solution (exemple PlantUML)

```plantuml
@startuml
!define RECTANGLE class
RECTANGLE "Sources de logs" as Sources
RECTANGLE "Agents d'ingestion (AMA, Logstash, Fluentd)" as Agents
RECTANGLE "Buffer/Streaming (Event Hub)" as Buffer
RECTANGLE "Analyse & Stockage (ADX, Log Analytics, Databricks, ELK)" as Analyse
RECTANGLE "Dashboards & Alerting (Power BI, Kibana, Azure Monitor)" as Dashboards

Sources --> Agents
Agents --> Buffer
Buffer --> Analyse
Analyse --> Dashboards
@enduml
```

## 2. Variantes selon les options

### a) Architecture ADX
```plantuml
@startuml
RECTANGLE "Sources de logs" as Sources
RECTANGLE "Azure Monitor Agent" as AMA
RECTANGLE "Event Hub" as EH
RECTANGLE "Azure Data Explorer (ADX)" as ADX
RECTANGLE "Power BI / KQL / Alerting" as BI

Sources --> AMA
AMA --> EH
EH --> ADX
ADX --> BI
@enduml
```

### b) Architecture ELK
```plantuml
@startuml
RECTANGLE "Sources de logs" as Sources
RECTANGLE "Filebeat / Logstash" as Logstash
RECTANGLE "Elasticsearch" as ES
RECTANGLE "Kibana" as Kibana

Sources --> Logstash
Logstash --> ES
ES --> Kibana
@enduml
```

### c) Architecture Databricks/ADLS
```plantuml
@startuml
RECTANGLE "Sources de logs" as Sources
RECTANGLE "Event Hub / Data Factory" as Ingestion
RECTANGLE "Azure Data Lake Storage (ADLS)" as ADLS
RECTANGLE "Azure Databricks" as DBX
RECTANGLE "Power BI / ML / Reporting" as BI

Sources --> Ingestion
Ingestion --> ADLS
ADLS --> DBX
DBX --> BI
@enduml
```

## 3. Légende et explications

- **Sources de logs** : applications, serveurs, appliances réseau
- **Agents d'ingestion** : outils pour collecter et transformer les logs
- **Buffer/Streaming** : absorption des pics, résilience
- **Analyse & Stockage** : moteur d'analyse, stockage hot/cold
- **Dashboards & Alerting** : visualisation, alertes, reporting

---
**Conclusion :**
Ces schémas illustrent les architectures cibles pour chaque solution. Ils peuvent être adaptés et enrichis selon les besoins spécifiques du client et les choix d'intégration.
