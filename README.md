# 📰 Persona — AI-Powered Personalized Newsletter

> Un agrégateur de news personnalisé piloté par l'IA, sans interface graphique. L'utilisateur interagit via un chatbot, configure ses préférences, et reçoit automatiquement une newsletter HTML à l'heure de son choix.

---

## 🏗️ Architecture

Le projet est composé de **3 workflows N8N** et d'une base de données **Supabase** :

```
┌─────────────────────────────────────────────────────────────────┐
│  Workflow A — Chatbot (Onboarding & Préférences)                │
│  Trigger: Chat Trigger (interface web intégrée N8N)            │
│  → Inscription / Connexion sécurisée                           │
│  → Configuration des préférences (topics, horaire, langue...)  │
│  → Authentification par session avec expiration                │
└─────────────────────────────────────────────────────────────────┘
                          │
                   [Supabase DB]
                          │
┌─────────────────────────────────────────────────────────────────┐
│  Workflow B — Newsletter (Génération & Envoi)                   │
│  Trigger: Schedule (toutes les heures)                         │
│  → Récupération des utilisateurs actifs                        │
│  → Vérification de l'horaire selon timezone de chaque user     │
│  → Agrégation news: NewsAPI + The Guardian + RSS BBC           │
│  → Filtrage & résumé par LLM (OpenRouter)                      │
│  → Email HTML personnalisé envoyé via Gmail                    │
│  → Log en base de données                                      │
└─────────────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────────────┐
│  Workflow C — Désinscription (RGPD)                            │
│  Trigger: Webhook GET /unsubscribe                             │
│  → Vérification du token sécurisé                              │
│  → Anonymisation email + désactivation compte                  │
│  → Suppression préférences & sessions                          │
│  → Page de confirmation HTML                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🚀 Prérequis

- [N8N](https://n8n.io/) installé localement (`npx n8n` ou Docker)
- Un compte [Supabase](https://supabase.com/) (gratuit)
- Un compte [OpenRouter](https://openrouter.ai/) pour le LLM (gratuit avec quota)
- Un compte [NewsAPI](https://newsapi.org/) (gratuit)
- Un compte [The Guardian API](https://open-platform.theguardian.com/) (gratuit)
- Un compte Gmail avec OAuth2 configuré dans N8N

---

## 🗄️ Base de données Supabase

Créez les 4 tables suivantes dans l'**SQL Editor** de Supabase :

```sql
-- Utilisateurs
CREATE TABLE users (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username          TEXT UNIQUE NOT NULL,
  password_hash     TEXT NOT NULL,
  email             TEXT NOT NULL,
  created_at        TIMESTAMP DEFAULT now(),
  is_active         BOOLEAN DEFAULT true,
  gdpr_consent      BOOLEAN DEFAULT false,
  gdpr_consent_date TIMESTAMP
);

-- Préférences newsletter
CREATE TABLE preferences (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id) ON DELETE CASCADE,
  topics        TEXT[],
  schedule_time TIME NOT NULL DEFAULT '08:00:00',
  schedule_tz   TEXT DEFAULT 'UTC',
  custom_prompt TEXT,
  language      TEXT DEFAULT 'fr',
  num_articles  INT DEFAULT 5
);

-- Logs d'envoi
CREATE TABLE newsletter_logs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
  newsletter_id   VARCHAR(255),
  sent_at         TIMESTAMP DEFAULT now(),
  articles        JSONB,
  feedback        TEXT,
  feedback_source VARCHAR(50) CHECK (feedback_source IN ('chat', 'email_link')),
  rating          VARCHAR(50) CHECK (rating IN ('good', 'bad')),
  created_at      TIMESTAMP DEFAULT now(),
  updated_at      TIMESTAMP DEFAULT now()
);

-- Sessions chatbot
CREATE TABLE sessions (
  session_id TEXT PRIMARY KEY,
  user_id    UUID REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT now(),
  expires_at TIMESTAMP
);
```

---

## ⚙️ Installation et configuration

### 1. Cloner le projet

```bash
git clone <url-du-repo>
cd persona-agent
```

### 2. Configurer les credentials dans N8N

Avant d'importer les workflows, créez les credentials suivants dans **N8N > Settings > Credentials** :

| Credential | Type | Où l'obtenir |
|---|---|---|
| `Supabase account` | Supabase API | Supabase > Project Settings > API > `service_role` key |
| `OpenRouter API` | HTTP Header Auth | [openrouter.ai/keys](https://openrouter.ai/keys) |
| `Gmail account` | Gmail OAuth2 | Google Cloud Console > OAuth 2.0 |

### 3. Importer les workflows

Dans N8N, pour **chaque fichier JSON** :
1. Cliquez sur **+** (nouveau workflow)
2. Cliquez sur les **3 points** en haut à droite > **Import from file**
3. Sélectionnez le fichier correspondant

**Ordre d'import recommandé :**
1. `Persona_workflow_C.json` — Désinscription
2. `Persona_workflow_A.json` — Chatbot
3. `Persona_workflow_B.json` — Newsletter

### 4. Configurer les clés API dans les nœuds

Dans le **Workflow B**, mettez à jour les nœuds suivants avec vos clés :

| Nœud | Paramètre | Valeur |
|---|---|---|
| `NewsAPI` | `apiKey` | Votre clé NewsAPI |
| `The Guardian API` | `api-key` | Votre clé Guardian (ou `test`) |

Dans le **Workflow B — nœud "Construire email HTML"**, mettez à jour les liens :

```javascript
// Lien de désinscription (URL du Webhook C en production)
http://localhost:5678/webhook/unsubscribe

// Lien de modification des préférences (URL du Chat Trigger A)
http://localhost:5678/webhook/persona-chatbot-webhook-001/chat

// Liens de feedback email vers le Webhook feedback
http://localhost:5678/webhook/feedback?uid={{ userId }}&token={{ token }}&newsletter_id={{ newsletterId }}&rating=good
http://localhost:5678/webhook/feedback?uid={{ userId }}&token={{ token }}&newsletter_id={{ newsletterId }}&rating=bad
```

### 5. Activer les workflows

**Dans cet ordre impératif :**

1. **Workflow C** → Cliquer sur **Publish** (pour activer l'URL de désinscription)
2. **Workflow A** → Activer "Make Chat Publicly Available" puis **Publish**
3. **Workflow B** → **Publish** (il se déclenchera automatiquement toutes les heures)

---

## 🧪 Tester le projet

### Test complet de bout en bout

**Étape 1 — Inscription via le chatbot**
1. Ouvrez `http://localhost:5678/webhook/persona-chatbot-webhook-001/chat`
2. Dites "Bonjour" → l'agent propose inscription ou connexion
3. Inscrivez-vous avec un `username`, un `email` et un `mot de passe`
4. Configurez vos préférences :
   - Topics : ex. `technologie, sport`
   - Horaire : ex. `20:00`
   - Timezone : ex. `Africa/Abidjan`
   - Langue : `fr` ou `en`
   - Nombre d'articles : ex. `3`

**Étape 2 — Envoi de la newsletter**

Pour tester sans attendre l'heure programmée :
1. Allez dans le **Workflow B**
2. Vérifiez que votre `schedule_time` dans Supabase correspond à l'heure actuelle (ou modifiez-la temporairement)
3. Cliquez sur **Execute Workflow**
4. Vérifiez votre boîte email !

**Étape 3 — Test de désinscription**
1. Ouvrez l'email reçu
2. Cliquez sur **"Se désinscrire"** en bas de l'email
3. Une page "✅ Désinscription confirmée" doit s'afficher
4. Vérifiez dans Supabase :
   - `users` → email anonymisé (`deleted_...`), `is_active = false`
   - `preferences` → ligne supprimée
   - `sessions` → sessions supprimées

---

## 📁 Structure du projet

```
.
├── Persona_workflow_A.json   # Workflow Chatbot (Onboarding & Auth)
├── Persona_workflow_B.json   # Workflow Newsletter (Génération & Envoi)
├── Persona_workflow_C.json   # Workflow Désinscription (RGPD)
├── SETUP.md                  # Guide de mise en service rapide
├── README.md                 # Ce fichier
└── .env                      # Variables d'environnement (ne pas committer !)
```

---

## 🔒 Sécurité & RGPD

### Authentification
- Les mots de passe sont **hashés** via SHA-256 avant stockage en base de données.
- Les sessions ont une **durée d'expiration** configurée (24h par défaut).
- Chaque session est identifiée par un `session_id` unique généré par N8N.
- Le chatbot refuse toute action si la session est invalide ou expirée.

### Liens de désinscription & feedback
- Chaque lien contient un `uid` et un `token` de sécurité calculé par un **hash bit-shift JS** avec un sel secret (`CHANGE_ME_SECRET_SALT`). Note : le module `crypto` (HMAC) n'étant pas disponible par défaut dans les nœuds Code de N8N, cette implémentation est suffisante pour un usage académique.
- Le **Workflow C** revalide le token avant toute suppression de données ou enregistrement de feedback.
- Un lien invalide affiche une page d'erreur sans effectuer aucune action.
- Les clics sur **"J'ai aimé" / "Je n'ai pas aimé"** dans l'email font un **UPDATE** sur la ligne existante dans `newsletter_logs` (filtrée par `newsletter_id`), évitant ainsi les doublons.

### Conformité RGPD
- L'utilisateur peut se désinscrire à tout moment via le lien dans chaque email.
- La désinscription **anonymise** l'email (ne le supprime pas, pour conserver l'intégrité référentielle) et désactive le compte.
- Les préférences et sessions sont **supprimées définitivement** lors de la désinscription.

---

## 🧠 Fonctionnement du LLM

Le **Workflow B** utilise un LLM (via OpenRouter) pour :
1. **Filtrer** les articles parmi les sources selon les topics de l'utilisateur
2. **Résumer** chaque article sélectionné en 2-3 phrases dans la langue de l'utilisateur
3. **Retourner** un JSON structuré avec exactement `num_articles` articles

Le prompt envoyé à l'IA est enrichi dynamiquement avec :
- **`custom_prompt`** : instructions de style définies par l'utilisateur via le chatbot (ex : _"Adopte un ton humoristique"_, _"Utilise des bullet points"_). Stocké dans `preferences.custom_prompt`.
- **Feedbacks passés** : les avis laissés sur les newsletters précédentes (via le chatbot ou les boutons dans l'email) sont récupérés depuis `newsletter_logs` et injectés dans le prompt pour affiner la sélection du lendemain. Ce mécanisme s'exécute de manière **séquentielle** avant les appels API pour garantir la disponibilité des données.
- **`factCheckScore`** : chaque article reçoit un score de confirmation croisée entre sources. L'IA est instruite de prioriser les articles avec un score élevé.

Le modèle par défaut est configurable dans le nœud **"LLM Newsletter"**. Le projet utilise un modèle gratuit via OpenRouter, mais peut être facilement switché vers GPT-4, Claude, Mistral, etc.

---

## 📡 Sources d'actualités

| Source | Type | Langue |
|---|---|---|
| [NewsAPI](https://newsapi.org/) | API REST | Multi-langue |
| [The Guardian](https://open-platform.theguardian.com/) | API REST | Anglais |
| [RSS BBC News](https://feeds.bbci.co.uk/news/rss.xml) | Feed RSS | Anglais |

---

## 🌍 Support des timezones

Le système supporte toutes les timezones standard, y compris les fuseaux africains :

```
Africa/Abidjan, Africa/Dakar, Africa/Casablanca,
Africa/Lagos, Africa/Nairobi, Africa/Johannesburg...
```

La comparaison d'horaire se fait au niveau de l'heure (format `HH`) pour éviter les problèmes de décalage à la minute près.

---

## 🛠️ Technologies utilisées

| Outil | Usage |
|---|---|
| **N8N** | Orchestration des workflows |
| **Supabase** | Base de données PostgreSQL + API REST |
| **OpenRouter** | Accès unifié aux LLMs (gratuit avec quota) |
| **NewsAPI** | Agrégation d'actualités |
| **The Guardian API** | Source d'actualités premium |
| **Gmail OAuth2** | Envoi des newsletters |

---

## 👥 Équipe

Projet réalisé dans le cadre du cursus **Ridwane777 PGE2 — G-AIA-410**.
