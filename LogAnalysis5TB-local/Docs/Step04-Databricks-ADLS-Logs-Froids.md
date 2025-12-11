
# Step 4 : Databricks/ADLS pour logs froids

## Objectif
Explorer l'utilisation de Databricks et Azure Data Lake Storage (ADLS) pour l'archivage long terme et l'analyse massive de logs froids.

## 1. Architecture cible

- **Sources** : Export de logs depuis ADX, Log Analytics, appliances, applications
- **Stockage** : Azure Data Lake Storage Gen2 (ADLS) pour l'archivage longue durée (plusieurs Po possibles)
- **Traitement** : Azure Databricks (Spark) pour l'analyse batch, l'exploration, le machine learning
- **Accès** : Notebooks Databricks, Power BI, API, jobs automatisés

![Architecture Databricks-ADLS](https://learn.microsoft.com/fr-fr/azure/databricks/_static/images/architecture/architecture-overview.png)

## 2. Scénarios d'analyse a posteriori

- Recherche d'incidents sur de longues périodes (compliance, forensic)
- Analyses statistiques massives (tendances, corrélations, ML)
- Requêtes ad hoc sur de très grands volumes (logs froids)
- Ré-entraînement de modèles de détection d'anomalies

## 3. Coût et performance

- **Stockage ADLS** : très compétitif pour l'archivage (facturation à la capacité, tiering possible)
- **Databricks** : coût à l'usage (clusters à la demande, auto-pause), dimensionnement selon la volumétrie et la fréquence d'analyse
- **Performance** : très élevée pour le traitement batch/distribué, moins adaptée à l'interactif temps réel

## 4. Complémentarité avec ADX/Log Analytics

- Utiliser ADX/Log Analytics pour l'ingestion, l'alerting, l'analyse rapide sur logs chauds
- Exporter les logs froids vers ADLS pour archivage et analyses massives a posteriori
- Databricks permet d'exploiter la donnée archivée pour des analyses avancées, du ML, ou des besoins réglementaires

---
**Conclusion :**
La combinaison Databricks + ADLS est idéale pour l'archivage long terme et l'analyse massive de logs froids. Elle complète parfaitement ADX/Log Analytics, en offrant une solution scalable, économique et puissante pour l'exploration et la valorisation des données historiques.
