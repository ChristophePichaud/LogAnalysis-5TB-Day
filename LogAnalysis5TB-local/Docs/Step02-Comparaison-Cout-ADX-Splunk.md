
# Step 2 : Comparaison des coûts ADX vs Splunk

## Objectif
Comparer le coût estimé d'un cluster Azure Data Explorer (ADX) traitant 5 To/jour avec les coûts typiques d'une solution Splunk équivalente.

## 1. Méthodologie d'estimation des coûts

L'estimation s'appuie sur :
- Les calculateurs officiels Azure et Splunk
- Les modèles de tarification publics (PaaS pour ADX, licence/consommation pour Splunk)
- Les hypothèses de volume, rétention, et usage

## 2. Hypothèses de dimensionnement

- **Volume de logs** : 5 To/jour (ingestion)
- **Rétention** : 30 jours (hot), 365 jours (cold)
- **Utilisateurs** : 20 analystes, 5 administrateurs
- **Requêtes** : 80% analytiques, 20% alerting
- **Région Azure** : Europe Ouest

## 3. Simulation de coût ADX

| Élément                | Hypothèse                  | Coût mensuel estimé (EUR) |
|------------------------|----------------------------|---------------------------|
| Cluster ADX (D14_v2)   | 2 nœuds, 30 jours, 24/7    | ~7 500 €                  |
| Stockage hot (30j)     | 150 To                     | ~1 500 €                  |
| Stockage cold (ADLS)   | 1,7 Po (365j)              | ~2 000 €                  |
| Réseau, monitoring     | Divers                     | ~500 €                    |
| **Total ADX**          |                            | **~11 500 €**             |

> **Remarque** : Les coûts peuvent varier selon le dimensionnement, la compression, la région et les optimisations d'ingestion.

## 4. Simulation de coût Splunk

| Élément                | Hypothèse                  | Coût mensuel estimé (EUR) |
|------------------------|----------------------------|---------------------------|
| Licence Splunk         | 5 To/jour (ingest)         | ~25 000 €                 |
| Infrastructure         | VM, stockage, support      | ~5 000 €                  |
| Maintenance/Support    | Divers                     | ~2 000 €                  |
| **Total Splunk**       |                            | **~32 000 €**             |

> **Remarque** : Les coûts Splunk sont très sensibles au volume ingéré et à la politique de licence. Les offres cloud Splunk peuvent être plus chères.

## 5. Synthèse comparative

| Solution   | Coût mensuel estimé | Points forts                  | Points faibles                |
|------------|--------------------|-------------------------------|-------------------------------|
| ADX        | ~11 500 €          | Scalabilité, coût, cloud natif| Courbe KQL, dépendance Azure  |
| Splunk     | ~32 000 €          | Maturité, écosystème, support | Coût, scalabilité limitée     |

**Conclusion :**
Azure Data Explorer (ADX) offre un avantage financier significatif pour de très gros volumes de logs, tout en restant performant et flexible. Splunk reste une référence en termes de fonctionnalités, mais son coût devient prohibitif à grande échelle.
