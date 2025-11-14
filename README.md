github-actions-examples

- 1 Create a project on GitHub (maven java/springboot)
- 2 Create a workflow GitHub Actions
- 3 rapport sonar of my project (in sonarQube)
- 4 Astuce pro : SonarCloud peut décorer directement
- 5 tes PRs avec les résultats d’analyse, ce qui est très pratique pour la revue de code.
- 6 OWASP Dependency check Sécurité des dépendances détecte vulnérabilités.
- 7 éventuellement publication artefact =>Nexus
- Tu peux conditionner certaines étapes uniquement pour master, par exemple publication vers Nexus ou Docker Hub :
- if: github.ref == 'refs/heads/master'
- Pour develop, tu peux faire analyse stricte Sonar + tests + couverture mais pas de publication.
- créer un workflow GitHub Actions complet “master vs develop” avec :
- Build, tests, Jacoco coverage, Sonar pour develop
- Build, tests, Jacoco coverage, Sonar + artefact/Docker pour master
- sonar -> PR Decoration : SonarCloud peut commenter directement sur la PR les problèmes de code
- Exemple Maven avec Jacoco pour Sonar :
- https://sonarcloud.io/summary/overall?id=souleymanebarry_demo-ci-cd&branch=master
- SONAR_TOKEN = d564aee63f36fe0850556cfb4d1154bc7adb994a


🧭 Explication du schéma d’architecture micro-services

L’architecture présentée illustre un système de gestion de commandes distribué
selon une approche micro-services:
où chaque service est autonome, découplé, et communique via un bus d’événements Kafka.
Les échanges inter-services sont asynchrones, ce qui améliore la scalabilité, la résilience, et la tolérance aux pannes.

**🏛️ 1. Les micro-services principaux**
1. Order Service

- Reçoit une nouvelle commande du client.

- Publie un événement order.placed vers Kafka.

- Suit l’avancement de la commande en consommant les événements stock.checked et payment.result.

- Publie ensuite order.confirmed ou order.failed.

2. Stock Service

- Consomme order.placed.

- Vérifie la disponibilité des produits.

- Publie stock.checked (stock OK ou insuffisant).

- Peut aussi recevoir une commande de compensation stock.release

3. Billing Service

- Consomme payment.request.

- Traite le paiement (ou appelle un prestataire externe).

- Publie payment.result (succès ou échec).

4. Notification Service

- Consomme order.confirmed ou order.failed.

- Envoie un email, SMS ou push au client.

**📡 2. Le rôle central de Kafka**

Kafka agit comme bus d’événements, assurant :

- Découplage fort entre services (aucun appel direct).

- Résilience (même si un service est indisponible, les messages persistent).

- Rejeu possible grâce à la rétention des topics.

- Scalabilité horizontale via les Consumer Groups.

- Chaque événement métier est publié dans un topic dédié, par exemple :

| Service      | Produit                                           | Consomme                          |
| ------------ | ------------------------------------------------- | --------------------------------- |
| Order        | `order.placed`, `order.confirmed`, `order.failed` | `stock.checked`, `payment.result` |
| Stock        | `stock.checked`                                   | `order.placed`, `stock.release`   |
| Billing      | `payment.result`                                  | `payment.request`                 |
| Notification | -                                                 | `order.confirmed`, `order.failed` |

🔄 3. **Orchestration Saga : gestion du workflow et compensations**
Pour garantir la cohérence de la commande, l’architecture repose sur une Saga.
Deux approches existent : choreography et orchestration.
Ici, c’est l’orchestrateur qui pilote la progression.

# Étapes de la saga (flux nominal)

- order.placed

- Orchestrator → stock.check.request

- stock.checked

- Orchestrator → payment.request

- payment.result

- Orchestrator → order.confirmed

# Gestion des erreurs

## ❌ Stock insuffisant

L'Orchestrator reçoit stock.checked (available = false).

Il publie :
→ order.failed

Aucune action de paiement n’est lancée.

## ❌ Paiement échoué

L'Orchestrator reçoit payment.result (status = failed).

Il publie d’abord une compensation :
→ stock.release (libération des réservations).

Puis il publie :
→ order.failed.

🛡️ 4. Résilience, fiabilité et bonnes pratiques
Idempotence

Chaque service doit pouvoir traiter un même message plusieurs fois sans effet secondaire grâce à :

des clés idempotentes (orderId, idempotencyKey)

un journal des messages déjà consommés.

Retries & DLQ

Des tentatives automatiques sont configurées.

Après plusieurs échecs, les messages sont envoyés dans une Dead Letter Queue (dlq) pour inspection manuelle.

Observabilité

Traces distribuées (OpenTelemetry)

Corrélation avec correlationId et orderId

Dashboards Kafka (lag, throughput, erreurs)

🧩 5. Résumé visuel et conceptuel

En résumé, l'architecture présente :

✔️ Une chaîne de traitement totalement asynchrone

Chaque service communique via Kafka, assurant la souplesse et la tolérance aux pannes.

✔️ Une orchestraton Saga pour garantir la cohérence

L’orchestrateur décide des actions à effectuer et publie les commandes et compensations.

✔️ Une responsabilisation claire des services

Chaque service ne fait qu’une seule chose, mais de manière fiable et scalable.

✔️ Un mécanisme de compensation robuste

Permet d'éviter les incohérences en cas d’échec partiel.
