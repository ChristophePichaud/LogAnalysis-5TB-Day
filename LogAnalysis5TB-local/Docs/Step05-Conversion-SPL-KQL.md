
# Step 5 : Conversion SPL vers KQL

## Objectif
Examiner la méthodologie de conversion des requêtes Splunk SPL vers KQL pour faciliter la formation des analystes.

## 1. Principales différences SPL/KQL

- **Syntaxe** : SPL (Splunk Processing Language) est orienté pipeline, KQL (Kusto Query Language) est orienté SQL-like avec des opérateurs enchaînés.
- **Fonctions** : Les fonctions d'agrégation, de parsing, de stats et de recherche diffèrent dans leur nommage et leur usage.
- **Gestion du temps** : Les deux langages gèrent le time slicing, mais la syntaxe diffère.
- **Visualisation** : SPL intègre la visualisation, KQL s'appuie sur des outils externes (Power BI, dashboards Azure).

## 2. Outils et guides de conversion

- Documentation officielle Microsoft : [Guide de migration SPL vers KQL](https://learn.microsoft.com/fr-fr/azure/data-explorer/kusto/query/spl)
- Outils communautaires de conversion (scripts Python, plugins VSCode)
- Tableaux de correspondance SPL/KQL (opérateurs, fonctions)

## 3. Exemples concrets

| SPL (Splunk)                                   | KQL (ADX)                                      |
|------------------------------------------------|------------------------------------------------|
| `index=logs error | stats count by host`        | `logs | where error == true | summarize count() by host` |
| `... | timechart span=1h count`                | `... | summarize count() by bin(TimeGenerated, 1h)` |
| `... | top 10 user`                            | `... | summarize count() by user | top 10 by count_` |

## 4. Plan de formation

- Sensibilisation aux concepts KQL (opérateurs, pipes, agrégations)
- Ateliers de conversion de requêtes SPL -> KQL sur des cas réels
- Mise à disposition de guides de correspondance et d'exemples
- Accompagnement sur les outils d'exploration (portail Azure, notebooks, Power BI)

---
**Conclusion :**
La conversion SPL vers KQL nécessite un accompagnement méthodologique et pratique. L'effort de formation est limité pour des analystes déjà familiers avec les requêtes Splunk, grâce à la richesse de la documentation et des outils d'aide à la migration.
