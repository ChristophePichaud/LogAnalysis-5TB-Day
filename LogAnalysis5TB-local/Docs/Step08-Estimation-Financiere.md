
# Step 8 : Estimation financière des solutions

## Objectif
Estimer le coût global des trois solutions proposées (ADX, ELK, Databricks).

## 1. Hypothèses de calcul

- **Volume de logs** : 5 To/jour
- **Rétention** : 30 jours (hot), 365 jours (cold)
- **Utilisateurs** : 20 analystes, 5 administrateurs
- **Région** : Europe Ouest
- **Support 24/7** inclus

## 2. Estimation détaillée par solution

### Azure Data Explorer (ADX)

| Élément                | Hypothèse                  | Coût mensuel estimé (EUR) |
|------------------------|----------------------------|---------------------------|
| Cluster ADX (D14_v2)   | 2 nœuds, 30 jours, 24/7    | ~7 500 €                  |
| Stockage hot (30j)     | 150 To                     | ~1 500 €                  |
| Stockage cold (ADLS)   | 1,7 Po (365j)              | ~2 000 €                  |
| Réseau, monitoring     | Divers                     | ~500 €                    |
| **Total ADX**          |                            | **~11 500 €**             |

### ELK Stack (self-managed sur Azure)

| Élément                | Hypothèse                  | Coût mensuel estimé (EUR) |
|------------------------|----------------------------|---------------------------|
| VM Elasticsearch       | 6 nœuds E8s_v4, 24/7       | ~7 000 €                  |
| Stockage SSD (100 To)  | Premium SSD                | ~2 000 €                  |
| Support, maintenance   | Divers                     | ~1 000 €                  |
| **Total ELK**          |                            | **~10 000 €**             |

### Databricks + ADLS

| Élément                | Hypothèse                  | Coût mensuel estimé (EUR) |
|------------------------|----------------------------|---------------------------|
| Stockage ADLS (1,7 Po) | Archivage long terme       | ~2 000 €                  |
| Clusters Databricks    | 2 jobs batch/semaine       | ~3 000 €                  |
| Support, monitoring    | Divers                     | ~500 €                    |
| **Total Databricks**   |                            | **~5 500 €**              |

## 3. Synthèse comparative

| Solution   | Coût mensuel estimé | Points forts                  | Points faibles                |
|------------|--------------------|-------------------------------|-------------------------------|
| ADX        | ~11 500 €          | Scalabilité, cloud natif, KQL | Courbe KQL, dépendance Azure  |
| ELK        | ~10 000 €          | Contrôle, open source         | Maintenance, tuning           |
| Databricks | ~5 500 €           | Coût stockage, ML, batch      | Moins interactif, batch only  |

---
**Conclusion :**
Chaque solution présente des avantages et des coûts spécifiques. ADX est optimal pour l'analyse interactive et l'intégration Azure, ELK pour le contrôle open source, Databricks/ADLS pour l'archivage et l'analyse massive à coût optimisé.
