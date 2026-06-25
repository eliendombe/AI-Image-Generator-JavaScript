# Documentation générale — AI Image Generator (français)

Bienvenue dans la documentation générale du projet "AI-Image-Generator-JavaScript". Ce document centralise les objectifs du projet, l'architecture globale, le diagramme de flux, le guide d'installation (résumé), une checklist de préparation pour la production, et des exemples de code pour intégrer un backend et persistance.

Table des matières
- Contexte et objectifs
- Structure du dépôt
- Architecture (résumé) — voir ARCHITECTURE.md
- Diagramme de flux (général)
- Guide d'installation (résumé) — voir README.md pour le guide d'installation détaillé
- Checklist de production
- Exemples d'intégration backend / persistance
- Variables d'environnement recommandées
- Sécurité et bonnes pratiques


Contexte et objectifs
---------------------
Ce projet fournit une interface front-end (HTML/CSS/JS) pour générer des images à partir de prompts en utilisant des services d'IA (API externes comme OpenAI, Stability AI, etc.). Le dépôt contient principalement le client (UI) et des exemples d'intégration backend pour la persistance et la sécurisation des clés API.

Structure du dépôt
------------------
- index.html / pages : interface utilisateur
- assets / css / styles : styles et composants visuels
- src / js : logique front-end (requêtes vers API de génération, preview, gestion d'upload)
- examples/backend : exemples d'API Node/Express pour proxy et persistance
- docs : documentation (ARCHITECTURE.md, DOCUMENTATION_GENERALE.md, README.md)


Architecture (résumé)
---------------------
Voir le fichier ARCHITECTURE.md pour un résumé détaillé de l'architecture. En bref :
- Frontend statique (JS/HTML/CSS)
- Backend optionnel (Node/Express) : proxy des appels vers l'API d'IA, validation, persistance
- Stockage : fichiers (local), ou objet (S3), et base de données pour métadonnées (Postgres/Mongo)
- Workers (optionnel) pour tâches longues


Diagramme de flux (général)
---------------------------
Voici un diagramme de flux simple en ASCII/Mermaid (si rendu pris en charge) :

```mermaid
flowchart TD
  A[Utilisateur : UI] -->|prompt| B(Frontend JS)
  B -->|POST /api/generate| C[Backend API (optionnel)]
  C -->|Appel REST| D{API IA externe}
  D -->|image base64 / URL| C
  C -->|enregistre| E[Stockage (S3 / local)]
  C -->|métadonnées| F[(Base de données)]
  C -->|réponse| B
  B -->|affiche| G[Galerie / Téléchargement]
```

(Explication : le frontend peut appeler directement l'API d'IA si la clé est publique/protégée ou si l'utilisateur possède sa propre clé. Pour la plupart des déploiements producteurs, il est recommandé d'utiliser un backend pour protéger la clé et enregistrer les images.)


Guide d'installation (résumé)
-----------------------------
Pour le guide d'installation détaillé et commandes, consultez README.md. En bref :
1. Cloner le repo
2. Installer les dépendances (pour le backend exemple : npm install)
3. Configurer les variables d'environnement (API keys, DB, S3)
4. Lancer en local (frontend statique + backend en dev)


Checklist de production
-----------------------
- [ ] Ne pas exposer les clés API côté client
- [ ] Utiliser un backend pour proxy et limitation de débit (rate limiting)
- [ ] Stockage fiable (S3 ou équivalent)
- [ ] DB pour métadonnées (Postgres / MongoDB)
- [ ] Tests automatisés pour endpoints critiques
- [ ] Monitoring et alerting (erreurs, queue length)
- [ ] Sauvegardes régulières de la base de données
- [ ] CD/CI configuré pour déploiement (build & tests)
- [ ] Politique de purge des images si stockage coûteux


Exemples d'intégration backend / persistance
-------------------------------------------
Des snippets et exemples détaillés sont fournis plus bas (Express + sauvegarde locale / S3 + Postgres). Ces exemples sont conçus pour être copiés dans `examples/backend` et adaptés.


Variables d'environnement recommandées
-------------------------------------
- PORT=3000
- AI_API_KEY=… (clé du fournisseur IA si utilisé côté serveur)
- DATABASE_URL=postgresql://user:pass@host:port/db
- AWS_ACCESS_KEY_ID=…
- AWS_SECRET_ACCESS_KEY=…
- S3_BUCKET=nom-du-bucket


Sécurité et bonnes pratiques
---------------------------
- Ne jamais committer de clés — ajouter les patterns au .gitignore
- Valider et limiter la taille des prompts et images
- Mettre en place un système de quotas pour éviter les abus
- Logger mais respecter la vie privée des utilisateurs


Fichiers créés
---------------
- ARCHITECTURE.md — résumé architecture
- DOCUMENTATION_GENERALE.md — ce fichier
- README.md — guide d'installation détaillé


Si vous souhaitez que je crée aussi un dossier `examples/backend` avec les fichiers d'exemples (express server, script de persistance, dockerfile), dites-le et je l'ajouterai immédiatement.
