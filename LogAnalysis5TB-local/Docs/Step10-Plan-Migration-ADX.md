
# Step 10 : Plan de migration ADX

## Objectif
Structurer les étapes clés d'un plan de migration vers Azure Data Explorer (ADX).

## 1. Phasage du projet

1. **Cadrage** : analyse des besoins, volumétrie, exigences métiers
2. **Architecture cible** : définition de l'architecture ADX, choix des outils d'ingestion, sécurité
3. **Planification** : découpage en lots, jalons, ressources

## 2. Préparation des données

- Cartographie des sources de logs
- Nettoyage, normalisation et mapping des schémas
- Définition des stratégies de rétention (hot/cold)
- Mise en place des pipelines d'ingestion (Event Hub, Data Factory, etc.)

## 3. Migration des requêtes

- Inventaire des requêtes SPL existantes
- Conversion SPL → KQL (outils, guides, ateliers)
- Tests de performance et d'exactitude sur KQL

## 4. Tests et validation

- Tests d'ingestion (volumétrie, latence)
- Validation des requêtes et des alertes
- Recette fonctionnelle avec les analystes
- Tests de montée en charge

## 5. Formation et conduite du changement

- Sessions de formation KQL pour les analystes
- Documentation des nouveaux processus
- Support et accompagnement post-migration

---
**Conclusion :**
Un plan de migration structuré, itératif et accompagné est essentiel pour garantir le succès du passage à Azure Data Explorer, tout en minimisant les risques et en maximisant l'adoption par les équipes.
