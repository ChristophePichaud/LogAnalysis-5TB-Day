
# Step 6 : ELK Stack sur Azure

## Objectif
Étudier la faisabilité d'un déploiement ELK Stack (self-managed) sur Azure pour l'analyse de logs à grande échelle.

## 1. Architecture cible sur Azure

- **Ingestion** : Filebeat/Logstash déployés sur VM ou containers pour collecter et transformer les logs
- **Stockage/Indexation** : Cluster Elasticsearch (VMs Azure, AKS ou Azure Managed Elasticsearch)
- **Visualisation** : Kibana (VM, container ou service managé)
- **Sécurité** : Azure Firewall, Private Link, RBAC, chiffrement

![Architecture ELK sur Azure](https://learn.microsoft.com/fr-fr/azure/architecture/example-scenario/logging/media/elk-architecture.png)

## 2. Dimensionnement et coût

- **VMs** : 3-10 nœuds Elasticsearch selon la volumétrie (5 To/jour → 100+ To stockés)
- **Stockage** : Premium SSD/Ultra Disk pour la performance, Azure Blob pour l'archivage
- **Réseau** : Load balancer, VNet, monitoring
- **Coût** :
	- VM (E8s_v4 x 6, 24/7) : ~7 000 €/mois
	- Stockage (100 To SSD) : ~2 000 €/mois
	- Support, maintenance, licences : ~1 000 €/mois
	- **Total estimé** : **~10 000 €/mois** (hors surcharge d'exploitation)

## 3. Avantages et limites

### Avantages
- Contrôle total sur l'infrastructure et la configuration
- Large écosystème open source, plugins, communauté
- Flexibilité d'intégration avec d'autres outils

### Limites
- Complexité d'exploitation, maintenance, upgrades
- Scalabilité manuelle, tuning nécessaire
- Coût caché (exploitation, incidents, support)
- Moins d'intégration native avec Azure que ADX/Log Analytics

## 4. Comparaison avec ADX/Databricks

| Critère         | ELK Stack (Azure)      | ADX/Log Analytics         | Databricks/ADLS         |
|-----------------|-----------------------|--------------------------|-------------------------|
| Coût            | ~10 000 €/mois        | ~11 500 €/mois (ADX)     | Variable, stockage bas  |
| Scalabilité     | Manuelle              | Automatique              | Très élevée (batch)     |
| Maintenance     | À la charge du client | Managée Azure            | Managée (Databricks)    |
| Fonctionnalités | Très riche, plugins   | KQL, alerting natif      | ML, batch, exploration  |

---
**Conclusion :**
Le déploiement d'un ELK Stack sur Azure est pertinent pour les organisations souhaitant garder la main sur leur stack open source, mais il implique une charge d'exploitation et de maintenance importante. Pour des besoins de scalabilité, de simplicité et d'intégration cloud, ADX ou Databricks sont souvent plus adaptés.
