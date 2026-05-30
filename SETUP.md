# Persona — Guide de mise en service

## Avant d'importer les workflows

### 1. Tables Supabase à créer (SQL Editor)

```sql
-- Table utilisateurs
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username      TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  email         TEXT NOT NULL,
  created_at    TIMESTAMP DEFAULT now(),
  is_active     BOOLEAN DEFAULT true,
  gdpr_consent  BOOLEAN DEFAULT false,
  gdpr_consent_date TIMESTAMP
);

-- Table préférences
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

-- Table logs newsletter
CREATE TABLE newsletter_logs (
  id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id   UUID REFERENCES users(id),
  sent_at   TIMESTAMP DEFAULT now(),
  articles  JSONB,
  feedback  TEXT
);

-- Table sessions
CREATE TABLE sessions (
  session_id TEXT PRIMARY KEY,
  user_id    UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT now(),
  expires_at TIMESTAMP
);
```

---

## Import dans N8N

Pour chaque workflow (A, B, C) :
1. N8N → **+** (nouveau workflow) → **Import from file**
2. Sélectionne le fichier JSON correspondant

---

## Credentials à configurer (cherche "YOUR_" dans chaque JSON)

Remplace tous les placeholders dans chaque nœud concerné :

| Placeholder | Où le trouver |
|---|---|
| `YOUR_SUPABASE_CREDENTIAL_ID` | N8N > Credentials > Supabase (Service Role Key) |
| `YOUR_OPENROUTER_CREDENTIAL_ID` | openrouter.ai > Keys |
| `YOUR_GMAIL_CREDENTIAL_ID` | N8N > Credentials > Gmail OAuth2 |
| `YOUR_NEWSAPI_KEY` | newsapi.org (gratuit) |
| `YOUR_GUARDIAN_API_KEY` | open-platform.theguardian.com (gratuit) |
| `YOUR_N8N_BASE_URL` | L'URL de ton N8N local ou hébergé |
| `YOUR_CHATBOT_URL` | L'URL du Chat Trigger du Workflow A |
| `CHANGE_ME_SECRET_SALT` | Remplace par une chaîne secrète de ton choix (garde-la identique dans workflow B et C) |

---

## Ordre d'activation

1. **Workflow C** (Désinscription) → activer en premier pour avoir l'URL du webhook
2. **Workflow A** (Chatbot) → activer, copier l'URL du Chat Trigger
3. **Workflow B** (Newsletter) → activer en dernier

---

## Test rapide

1. Ouvre l'URL du Chat Trigger du Workflow A
2. Écris "Bonjour" → l'agent doit proposer inscription ou connexion
3. Inscris-toi avec un username, email, mot de passe
4. Reconnecte-toi dans une nouvelle conversation
5. Configure tes préférences (topics, horaire)
6. Pour tester l'envoi sans attendre l'heure : exécute le Workflow B manuellement

