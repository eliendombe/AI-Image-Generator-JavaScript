# AI-Image-Generator-JavaScript — Guide d'installation (FR)

Ce README explique comment cloner, configurer et exécuter le projet en local, comment déployer en production, et fournit des exemples d'intégration backend/persistance.

---

## Table des matières
- Pré-requis
- Installation rapide (mode développement)
- Variables d'environnement (.env exemple)
- Exécution (frontend seul / avec backend)
- Docker & Docker Compose
- Exemple d'API backend (Express) — intégration & persistance
- Migration de base de données (exemple Postgres)
- Checklist avant production
- FAQ

---

## Pré-requis
- Node.js (>= 18 LTS) et npm ou pnpm
- Git
- (Optionnel) Docker & Docker Compose
- (Optionnel) Compte S3 / MinIO pour stockage d'objets
- (Optionnel) Base de données Postgres ou MongoDB

---

## Installation rapide (local)
1. Cloner le dépôt :
   ```
   git clone https://github.com/eliendombe/AI-Image-Generator-JavaScript.git
   cd AI-Image-Generator-JavaScript
   ```

2. Installer dépendances (si vous utilisez l'exemple backend) :
   ```
   cd examples/backend
   npm install
   ```

3. Créer un fichier .env (voir section suivante) et remplir les variables.

4. Lancer le backend en dev :
   ```
   npm run dev
   ```
   (ou `node src/index.js` si pas de script)

5. Ouvrir le frontend : ouvrir `index.html` dans le navigateur ou servir le répertoire racine :
   ```
   npx serve .   # ou tout serveur statique
   ```

---

## Variables d'environnement (exemple .env)
```
# Serveur
PORT=3000

# API IA (clé côté serveur)
AI_PROVIDER=OpenAI
AI_API_KEY=sk-...

# Stockage S3
S3_ENDPOINT=https://s3.amazonaws.com
S3_BUCKET=my-bucket
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...

# Base de données (Postgres)
DATABASE_URL=postgresql://user:pass@localhost:5432/ai_images

# Queue (optionnel)
REDIS_URL=redis://localhost:6379
```

---

## Exécution — Scénarios

A. Frontend uniquement (pour tests rapides)
- Ouvrir index.html directement dans le navigateur.
- Attention : si l'intégration appelle l'API IA directement depuis le client, la clé sera exposée — déconseillé en production.

B. Frontend + Backend (recommandé)
- Lancer l'API backend (examples/backend).
- Le frontend effectue des requêtes vers /api/generate.

---

## Docker (exemple minimal)
Dockerfile (backend) :
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
CMD ["node", "dist/index.js"]
```

docker-compose.yml (backend + postgres + redis) :
```yaml
version: '3.8'
services:
  api:
    build: ./examples/backend
    ports:
      - "3000:3000"
    env_file: ./examples/backend/.env
    depends_on:
      - db
      - redis
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: ai_images
    volumes:
      - db-data:/var/lib/postgresql/data
  redis:
    image: redis:7
volumes:
  db-data:
```

---

## Exemple d'API backend (Express) — proxy + persistance (extrait)
Fichier : examples/backend/src/index.js

```javascript
import express from "express";
import fetch from "node-fetch";
import { Pool } from "pg";
import AWS from "aws-sdk";
import bodyParser from "body-parser";

const app = express();
app.use(bodyParser.json());

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

const s3 = new AWS.S3({
  accessKeyId: process.env.AWS_ACCESS_KEY_ID,
  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  endpoint: process.env.S3_ENDPOINT || undefined,
  s3ForcePathStyle: true,
});

app.post("/api/generate", async (req, res) => {
  const { prompt, options } = req.body;
  if (!prompt || prompt.length > 2000) return res.status(400).json({ error: "Prompt invalide" });

  try {
    // Exemple d'appel vers une API tierce (pseudo)
    const aiResp = await fetch("https://api.openai.com/v1/images", {
      method: "POST",
      headers: { "Authorization": `Bearer ${process.env.AI_API_KEY}`, "Content-Type": "application/json" },
      body: JSON.stringify({ prompt }),
    });
    const aiJson = await aiResp.json();
    // Supposons que aiJson contient une URL d'image
    const imageUrl = aiJson.data?.[0]?.url;

    // Enregistrer métadonnées en DB
    const result = await pool.query(
      `INSERT INTO images (prompt, status, image_url, provider_response, created_at) VALUES ($1,$2,$3,$4,NOW()) RETURNING id`,
      [prompt, 'ready', imageUrl, aiJson]
    );

    return res.json({ id: result.rows[0].id, image_url: imageUrl });
  } catch (err) {
    console.error(err);
    return res.status(500).json({ error: "Erreur serveur" });
  }
});

const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`API listening ${port}`));
```

Notes :
- Ici l'appel à l'API IA est synchrone; en production on préférera un job asynchrone.
- Pensez à gérer les erreurs et limites de l'API.

---

## Exemple : Upload vers S3 (Node)
```javascript
async function uploadBufferToS3(buffer, key, contentType="image/png") {
  const params = {
    Bucket: process.env.S3_BUCKET,
    Key: key,
    Body: buffer,
    ContentType: contentType,
    ACL: "private",
  };
  return s3.upload(params).promise();
}
```

---

## Schéma SQL minimal (Postgres)
```sql
CREATE TABLE images (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  prompt text NOT NULL,
  status text NOT NULL,
  image_url text,
  provider_response jsonb,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz
);

CREATE INDEX idx_images_created_at ON images(created_at DESC);
```

---

## Integration côté Frontend (extrait JS)
```javascript
async function generateImage(prompt) {
  const resp = await fetch("/api/generate", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ prompt })
  });
  if (!resp.ok) throw new Error("Erreur génération");
  return resp.json();
}
```

---

## Checklist avant production
- [ ] Ne pas exposer les clés API dans le client.
- [ ] Utiliser backend pour proxier et limiter l'accès (rate limits).
- [ ] Configurer monitoring & alerting (erreurs, latence).
- [ ] Tests automatisés (unitaires & intégration).
- [ ] Utiliser stockage objet résilient (S3) + CDN.
- [ ] Sauvegardes et rotation des logs.
- [ ] Politique de rétention / purge des images.
- [ ] Revue de la sécurité (pen-tests, dépendances).
- [ ] Mise en place de quotas/plan tarifaire si service public.

---

## Déploiement rapide (exemple)
1. Construire image Docker backend
   ```
   docker build -t ai-image-api ./examples/backend
   ```
2. Pousser l'image sur un registry (DockerHub, ECR).
3. Déployer via une plateforme (Kubernetes, ECS, Render, Heroku).
4. Configurer variables d'environnement et secrets.

---

## FAQ rapide
Q : Puis-je appeler directement OpenAI depuis le client ?
R : Techniquement oui, mais dangereux : la clé serait exposée. Utilisez le backend.

Q : Quel provider IA utiliser ?
R : OpenAI, Stability, Midjourney (selon usage/licences). Adaptez code d'appel.

---

Si vous voulez, je peux :
- Ajouter ces fichiers automatiquement au dépôt.
- Créer un dossier examples/backend complet (server, package.json, Dockerfile, README).
- Générer des scripts de migration (knex / sequelize / prisma).

Répondez "Ajouter au dépôt" pour que je les committe, ou dites quelles modifications vous voulez avant commit.
