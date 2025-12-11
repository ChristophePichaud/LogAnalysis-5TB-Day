# Besoin
Besoin d’analyse d’un solution d’analyse de logs avec 5 TB de données par jour
J'ai un besoin d'analyse de fichiers de logs pour une multinationale. Pour le moment, leur solution est basée sur Splunk. Le client veut passer sur une solution qui coute moins cher et veut éventuellement passer sur le Cloud Azure. Le client a évoqué Databricks. Mais il est ouvert à toute proposition. Tous les jours, il faut analyser 5 TB de fichiers de logs. Qu'en penses tu globalement et après on afinera.
Je veux faire une réponse dans un document d'Architecture. Je veux dfonc la création de plusieurs fichiers en mode markdown, des fichiers PUML d’architecture et le plan dans lequel je vais y insérer tous les documents et pages que tu vas me créer.
Voici les étapes (fichiers .md à créer). Nom du fichier à préfixer avec le numéro de Step.

# Etapes
Etapes de construction:
Step1: [ ] je veux un focus détaillé sur une solution  Azure Data Explorer (ADX) with KQL.
Step2 : [ ] comparions le coût estimé d'un cluster ADX gérant 5 TB/jour par rapport aux coûts typiques de Splunk
Step3 : [ ] creuser sur Azure Log Analytics pour les logs chauds (7-14 jours de rétention, alertes temps réel) 
Step4 : [ ] creuser sur Databricks/ADLS pour les logs froids (archivage long terme, analyses massives a posteriori)
Step5 : [ ] examinions la méthodologie de conversion de requêtes de Splunk SPL à KQL (pour former les analystes)
Step6 : [ ] creuser sur ELK Stack (self-managed) sur Azure
Step7 : [ ] faire une analyse plus détaillée des risques de la migration que cela soit ELK ou Azure ADX KQL 
Step8 : [ ] faire une estimation financière pour les 3 solutions proposées
Step9 : [ ] synthétisons tous ces points (Coût, Risque, Architecture) dans un tableau de bord final pour votre présentation client
Step10 : [ ] aide à structurer les étapes clés d'un plan de migration orienté ADX 
Step 11 : [ ] aide pour détailler les outils d'ingestion spécifiques pour remplacer les Universal Forwarders de Splunk dans cette architecture Azure
Step 12 : [ ] je souhaite un ou plusieurs schémas de type plantuml ou autre pour la partie Synthèse de la Solution : Bref aperçu de l'architecture cible.

