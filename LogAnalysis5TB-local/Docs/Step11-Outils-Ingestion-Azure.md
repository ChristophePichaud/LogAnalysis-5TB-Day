
# Step 11 : Outils d'ingestion pour Azure

## Objectif
Détailler les outils d'ingestion spécifiques pour remplacer les Universal Forwarders de Splunk dans l'architecture Azure.

## 1. Alternatives Azure aux Universal Forwarders

- **Azure Monitor Agent (AMA)** : agent natif pour collecter logs et métriques sur VM, serveurs, containers
- **Logstash** : pipeline open source, compatible Azure Event Hub, transformation et enrichissement des logs
- **Azure Event Hub** : service de streaming pour ingestion massive, bufferisation et distribution des logs
- **Azure Data Collector API** : ingestion personnalisée via API REST
- **Fluentd/Fluent Bit** : agents open source, support natif Azure, transformation flexible
- **Azure Data Factory** : ingestion batch, orchestration de flux de données

## 2. Architecture d'ingestion type

1. **Sources** : serveurs, applications, appliances réseau
2. **Agents** : AMA, Logstash, Fluentd, etc. déployés sur les sources
3. **Buffer/Streaming** : Event Hub pour absorber les pics et garantir la résilience
4. **Cibles** : Log Analytics, ADX, Data Lake, SIEM

![Architecture ingestion Azure](https://learn.microsoft.com/fr-fr/azure/architecture/example-scenario/logging/media/centralized-logging-architecture.png)

## 3. Bonnes pratiques

- Standardiser les formats de logs (JSON, CEF, Syslog)
- Sécuriser les flux (TLS, authentification, RBAC)
- Superviser l'état des agents et la latence d'ingestion
- Mettre en place des alertes sur les échecs d'ingestion
- Documenter les pipelines et les transformations

---
**Conclusion :**
L'écosystème Azure propose de nombreux outils pour remplacer les Universal Forwarders de Splunk, avec des solutions natives, open source et managées. Le choix dépendra du contexte technique, des volumes et des exigences de sécurité et de supervision.
