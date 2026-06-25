# ARCHITECTURE — AI Image Generator (Résumé)

Ce document décrit l'architecture du projet AI-Image-Generator-JavaScript : composants, flux de données, options de déploiement et recommandations pour la production.

## Vue d'ensemble
Le projet est organisé autour d'un frontend statique (HTML/CSS/JS) qui permet à l'utilisateur de saisir un prompt et d'afficher les images générées. Un backend optionnel (Node/Express) agit comme proxy vers l'API IA, effectue la validation, la persistance des métadonnées et l'enregistrement des images (local ou cloud).

Composants principaux :
- Frontend (client) : interface utilisateur, prévisualisation, envoi des prompts.
- Backend (optionnel mais recommandé) : API REST/GraphQL, gestion clés, orchestration requêtes IA, persistance.
- Stockage d'objets : S3 (ou compatible), stockage local pour développement.
- Base de données : Postgres / MongoDB pour les métadonnées (prompts, URL d'image, utilisateur, statut).
- Workers (optionnel) : traitement asynchrone des tâches longues (ex. génération via file d'attente).
- Observabilité : logs, métriques, alerting.

## Diagramme de flux (Mermaid)
Utilisez ce diagramme pour visualiser le flux principal :

```mermaid
flowchart TD
  U[Utilisateur] -->|saisie prompt| F(Frontend JS)
  F -->|POST /api/generate| B[Backend (Node/Express)]
  B -->|valide & auth| DB[(Base de données)]
  B -->|push job / sync call| Q[(Queue / Worker)] 
  Q -->|consume| W[Worker]
  B -->|call API IA (REST)| AI[API IA externe]
  AI -->|image (URL / base64)| B
  B -->|upload| S3[(Stockage Objet S3)]
  B -->|save metadata| DB
  B -->|réponse| F
  F -->|affiche| G[Galerie / Téléchargement]
```

Explication : le frontend peut appeler directement l'API IA uniquement si la clé n'est pas exposée. En production, route via backend.

## Modèle de données (exemple simplifié)
Table images (Postgres) :
- id (uuid)
- prompt (text)
- status (enum: pending, processing, ready, failed)
- image_url (text)
- thumbnail_url (text)
- provider_response (jsonb)
- created_at, updated_at
- user_id (nullable)

Table users (si nécessaire) :
- id, email, display_name, created_at

## Endpoints API (exemples)
- POST /api/generate
  - body: { prompt, options? }
  - response: { id, status }
- GET /api/images/:id
  - returns metadata and URLs
- GET /api/images?userId=...
  - liste paginée
- POST /api/upload (optionnel)
  - upload d'images utilisateur

## Flux synchrone vs asynchrone
- Synchrone : backend appelle l'API IA et renvoie l'image immédiatement — simple mais risque de timeouts et latence.
- Asynchrone (recommandé en production) : backend créé un job, répond immédiatement avec un id, worker génère l'image et met à jour DB + stockage.

## Sécurité & bonnes pratiques
- Ne jamais exposer les clés API côté client.
- Rate limiting côté backend.
- Validation côté serveur des prompts et taille des images.
- Mise en place d'authentification pour actions privées.
- Chiffrement des secrets (vault, variables d'environnement sécurisées).
- Sanitization des métadonnées stockées.

## Scalabilité
- Séparer workers et API pour scalabilité horizontale.
- Stocker images sur S3 + CDN pour distribution.
- Utiliser une DB relationnelle dimensionnée (indexation sur created_at, user_id).
- Mettre en cache les résultats si prompts fréquents.

## Observabilité & Sûreté
- Logs structurés (JSON), tracing des requêtes (opentracing / OpenTelemetry).
- Monitoring (Prometheus/Grafana) pour latence, erreurs, taux de réussite.
- Backups réguliers de la DB.
- Stratégie de purge ou lifecycle pour objet S3.

## Options de déploiement
- Simple : frontend statique sur Netlify / GitHub Pages, backend sur Heroku / Render, DB managed (Heroku Postgres).
- Containerisé : Docker + Docker Compose / Kubernetes. Workers en pods séparés.
- Infrastructure as Code recommandée pour production.

## Diagramme technique (components)
- Client (JS/HTML/CSS)
- API Server (Express)
- Job Queue (Redis + BullMQ / RabbitMQ)
- Worker(s)
- Storage (S3)
- DB (Postgres)
- CDN

## Notes finales
Ce résumé sert de base pour déployer en local ou production. Les décisions techniques (Postgres vs Mongo, queue system) dépendent des besoins : consistance, requêtes complexes et relations orientent vers Postgres, haute flexibilité JSON oriente vers MongoDB.
