
# Step 9 : Tableau de bord synthétique

## Objectif
Synthétiser les points clés (coût, risque, architecture) dans un tableau de bord pour la présentation client.

## 1. Tableau comparatif des solutions

| Critère         | ADX                    | ELK Stack              | Databricks/ADLS         |
|-----------------|------------------------|------------------------|-------------------------|
| Coût mensuel    | ~11 500 €              | ~10 000 €              | ~5 500 €                |
| Scalabilité     | Très élevée, managée   | Manuelle, tuning       | Très élevée (batch)     |
| Maintenance     | Managée Azure          | À la charge du client  | Managée (Databricks)    |
| Interactivité   | Temps réel, KQL        | Temps réel, DSL        | Batch, ML, exploration  |
| Sécurité        | Azure AD, RBAC, SIEM   | Custom, plugins        | Azure AD, RBAC          |
| Intégration     | Azure, Power BI, SIEM  | Open source, plugins   | Azure, ML, Power BI     |
| Risques         | Courbe KQL, dépendance | Exploitation, tuning   | Moins interactif        |

## 2. Recommandations

- **ADX** : recommandé pour l'analyse interactive, l'alerting temps réel, l'intégration Azure et la supervision massive.
- **ELK Stack** : pertinent pour les organisations souhaitant garder la main sur leur stack open source, avec une équipe d'exploitation expérimentée.
- **Databricks/ADLS** : idéal pour l'archivage long terme, l'analyse massive a posteriori, le machine learning et la conformité réglementaire.

## 3. Points d'attention

- Bien évaluer la courbe d'apprentissage KQL/SPL/DSL selon les équipes
- Anticiper les coûts cachés (exploitation, support, formation)
- Prévoir une phase pilote et des tests de migration
- Adapter la solution à la volumétrie réelle et aux besoins métiers

---
**Conclusion :**
Le choix de la solution dépendra des priorités (coût, interactivité, contrôle, conformité). Une architecture hybride (ADX pour le chaud, Databricks/ADLS pour le froid) est souvent optimale pour les très gros volumes de logs.
