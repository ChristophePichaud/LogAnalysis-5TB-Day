
# Step 3 : Azure Log Analytics pour logs chauds

## Objectif
Analyser l'intérêt d'Azure Log Analytics pour la gestion des logs chauds (rétention 7-14 jours, alertes temps réel).

## 1. Fonctionnalités principales

- Collecte centralisée de logs, métriques et traces depuis de multiples sources Azure et on-premises
- Analyse en temps réel via KQL (Kusto Query Language)
- Déclenchement d'alertes, automatisation (Logic Apps, Azure Monitor)
- Tableaux de bord interactifs, visualisation intégrée
- Intégration native avec Sentinel (SIEM), Application Insights, ADX

## 2. Architecture d'intégration

L'architecture type pour la gestion des logs chauds avec Log Analytics :

- **Sources** : VM, PaaS Azure, appliances, agents Log Analytics, Event Hub
- **Workspace Log Analytics** : point central de collecte et d'analyse
- **Alerting** : règles d'alerte, actions automatisées
- **Connecteurs** : export vers ADX, Power BI, SIEM, stockage externe

![Architecture Log Analytics](https://learn.microsoft.com/fr-fr/azure/azure-monitor/media/logs/log-analytics-architecture.png)

## 3. Scénarios d'usage

- Supervision opérationnelle (infrastructure, applicatif)
- Détection d'incidents et alerting temps réel
- Analyse rapide sur 7-14 jours de logs récents
- Investigation de sécurité (avec Azure Sentinel)

## 4. Limites et complémentarités avec ADX

### Limites
- Coût élevé pour de très gros volumes (>1 To/jour)
- Rétention limitée (max 2 ans, coût croissant)
- Moins adapté à l'analyse massive historique (préférer ADX ou Databricks pour le cold)

### Complémentarités
- Utiliser Log Analytics pour l'ingestion, l'alerting et l'analyse rapide sur logs chauds
- Exporter les données vers ADX pour l'analyse avancée, la rétention longue et l'exploration massive

---
**Conclusion :**
Azure Log Analytics est idéal pour la gestion des logs chauds, l'alerting temps réel et la supervision opérationnelle. Pour des besoins d'analyse massive ou de rétention longue, il est pertinent de le coupler à ADX ou à une solution de stockage/traitement Big Data.
