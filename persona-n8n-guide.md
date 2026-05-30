# Guide Complet — Projet Persona sur N8N
> Agrégateur de newsletters IA personnalisées | Première utilisation de N8N

---

## Table des matières

1. [Comprendre N8N en 5 minutes](#1-comprendre-n8n-en-5-minutes)
2. [Architecture globale du projet](#2-architecture-globale-du-projet)
3. [Supabase — structure de la base de données](#3-supabase--structure-de-la-base-de-données)
4. [MUST 1 — Chatbot trigger & authentification](#4-must-1--chatbot-trigger--authentification)
5. [MUST 2 — Onboarding utilisateur via le chatbot](#5-must-2--onboarding-utilisateur-via-le-chatbot)
6. [MUST 3 — Récupération des news depuis plusieurs sources](#6-must-3--récupération-des-news-depuis-plusieurs-sources)
7. [MUST 4 — Filtrage IA par utilisateur](#7-must-4--filtrage-ia-par-utilisateur)
8. [MUST 5 — Envoi de la newsletter par email](#8-must-5--envoi-de-la-newsletter-par-email)
9. [MUST 6 — Planification (schedule par utilisateur)](#9-must-6--planification-schedule-par-utilisateur)
10. [MUST 7 — RGPD de base & désinscription](#10-must-7--rgpd-de-base--désinscription)
11. [COULD 1 — Newsletter en HTML](#11-could-1--newsletter-en-html)
12. [COULD 2 — Commentaires sur les newsletters passées](#12-could-2--commentaires-sur-les-newsletters-passées)
13. [COULD 3 — Fact-checking](#13-could-3--fact-checking)
14. [COULD 4 — Prompt personnalisé par utilisateur](#14-could-4--prompt-personnalisé-par-utilisateur)
15. [Ordre de construction recommandé](#15-ordre-de-construction-recommandé)
16. [Erreurs fréquentes des débutants N8N](#16-erreurs-fréquentes-des-débutants-n8n)
17. [Checklist avant la démo](#17-checklist-avant-la-démo)

---

## 1. Comprendre N8N en 5 minutes

### Vocabulaire essentiel

| Terme | Définition |
|---|---|
| **Workflow** | Un enchaînement de nœuds, c'est ton "programme" |
| **Node (nœud)** | Une étape dans le workflow (ex: envoyer un email, interroger une API) |
| **Trigger** | Le nœud qui démarre le workflow (chatbot, cron, webhook...) |
| **Expression** | Une variable dynamique, syntaxe `{{ $json.champ }}` |
| **Credentials** | Tes clés API stockées de façon sécurisée dans N8N |
| **Execution** | Une instance d'exécution d'un workflow |
| **Item** | Un objet JSON qui circule entre les nœuds |

### Comment les données circulent

Chaque nœud reçoit un tableau d'**items** (objets JSON) et en retourne un nouveau. Pour accéder à la sortie du nœud précédent :

```
{{ $json.nom_du_champ }}           → valeur du champ
{{ $json["champ avec espace"] }}   → champ avec espace/caractère spécial
{{ $('Nom du nœud').item.json.x }} → valeur d'un nœud précis par son nom
```

### Les nœuds que tu utiliseras le plus

- **Chat Trigger** — point d'entrée chatbot
- **AI Agent** — cerveau IA avec mémoire de conversation
- **HTTP Request** — appel d'API externe (news, etc.)
- **Supabase** — lecture/écriture dans ta DB
- **Code (JavaScript)** — logique custom
- **IF / Switch** — branchements conditionnels
- **Send Email / Gmail** — envoi d'emails
- **Schedule Trigger** — déclenchement automatique à heure fixe
- **Set** — créer/modifier des champs dans tes items

---

## 2. Architecture globale du projet

Le projet se compose de **3 workflows distincts** :

```
┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW A — Chatbot (toujours actif)                       │
│                                                              │
│  Chat Trigger → AI Agent → Supabase (CRUD utilisateur)      │
│                          → Authentification                  │
│                          → Onboarding / Modification prefs  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW B — Envoi de newsletters (déclenché toutes les     │
│              heures ou toutes les X minutes)                 │
│                                                              │
│  Schedule Trigger → Supabase (qui doit recevoir maintenant?)│
│                   → HTTP Request (sources de news)          │
│                   → AI Agent (filtrage + résumé)            │
│                   → Send Email                              │
│                   → Supabase (log d'envoi)                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  WORKFLOW C — Désinscription (webhook ou lien email)         │
│                                                              │
│  Webhook Trigger → Supabase (suppression/désactivation)     │
│                  → Email de confirmation                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Supabase — structure de la base de données

### Tables à créer

#### Table `users`

```sql
CREATE TABLE users (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username     TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,          -- bcrypt ou sha256
  email        TEXT NOT NULL,
  created_at   TIMESTAMP DEFAULT now(),
  is_active    BOOLEAN DEFAULT true,    -- pour la désinscription RGPD
  gdpr_consent BOOLEAN DEFAULT false,
  gdpr_consent_date TIMESTAMP
);
```

#### Table `preferences`

```sql
CREATE TABLE preferences (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES users(id) ON DELETE CASCADE,
  topics        TEXT[],                 -- ex: ['tech', 'climat', 'sport']
  schedule_time TIME NOT NULL,          -- ex: '08:00:00'
  schedule_tz   TEXT DEFAULT 'UTC',
  custom_prompt TEXT,                   -- COULD 4
  language      TEXT DEFAULT 'fr',
  num_articles  INT DEFAULT 5
);
```

#### Table `newsletter_logs`

```sql
CREATE TABLE newsletter_logs (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID REFERENCES users(id),
  sent_at     TIMESTAMP DEFAULT now(),
  articles    JSONB,                    -- les articles envoyés
  feedback    TEXT                      -- COULD 2
);
```

#### Table `sessions`

```sql
CREATE TABLE sessions (
  session_id   TEXT PRIMARY KEY,        -- identifiant de la session chatbot
  user_id      UUID REFERENCES users(id),
  created_at   TIMESTAMP DEFAULT now(),
  expires_at   TIMESTAMP
);
```

### Connexion Supabase dans N8N

1. Dans N8N → **Settings > Credentials > New**
2. Cherche "Supabase"
3. Renseigne : URL du projet (`https://xxx.supabase.co`) + Service Role Key (dans Supabase > Settings > API)
4. Donne un nom clair : `Supabase - Persona`

> ⚠️ Utilise la **Service Role Key** (et non l'anon key) pour bypasser les Row Level Security depuis N8N. Garde-la secrète.

---

## 4. MUST 1 — Chatbot trigger & authentification

### Objectif

Permettre à un utilisateur de s'identifier via le chatbot, et bloquer tout accès non authentifié aux données personnelles.

### Nœuds à utiliser

```
[Chat Trigger] → [AI Agent] → [Code: extract intent] → [IF: logged in?]
                                                              ↓ Non
                                                         [Demander login/register]
                                                              ↓ Oui
                                                         [Suite du workflow]
```

### Étape par étape

**1. Ajouter le Chat Trigger**
- Ouvre un nouveau workflow
- Ajoute le nœud **"Chat Trigger"**
- Il génère automatiquement une URL de chatbot accessible dans ton navigateur
- Active l'option **"Allow file uploads"** si tu veux (optionnel)

**2. Stocker l'identité de session**

Le Chat Trigger fournit automatiquement un `sessionId` unique par conversation. C'est ta clé pour retrouver l'état d'un utilisateur.

```javascript
// Dans un nœud Code, récupère le sessionId :
const sessionId = $('Chat Trigger').item.json.sessionId;
```

**3. Logique d'authentification**

À chaque message reçu, avant de faire quoi que ce soit :

```
[Chat Trigger]
      ↓
[Supabase: SELECT * FROM sessions WHERE session_id = '{{sessionId}}']
      ↓
[IF: session trouvée et non expirée?]
    ↙ Oui                    ↘ Non
[Utilisateur connecté]    [AI Agent: demande login]
```

**4. Nœud d'enregistrement**

Quand l'utilisateur veut créer un compte :

```javascript
// Nœud Code — hachage du mot de passe (simple)
const crypto = require('crypto');
const password = $json.password;
const hash = crypto.createHash('sha256').update(password).digest('hex');
return [{ json: { password_hash: hash } }];
```

Puis un nœud **Supabase** en mode `INSERT` dans la table `users`.

**5. Nœud de login**

```javascript
// Nœud Code — vérification
const inputHash = crypto.createHash('sha256').update($json.password).digest('hex');
const storedHash = $('Supabase: get user').item.json.password_hash;
const isValid = inputHash === storedHash;
return [{ json: { isValid } }];
```

Si valide → INSERT dans `sessions` avec `expires_at = now() + 24h`.

### Test de sécurité à faire

- [ ] Essayer d'accéder aux données d'un autre utilisateur en changeant manuellement l'email dans le chat
- [ ] Vérifier qu'un session_id expiré ne donne plus accès
- [ ] Vérifier qu'un utilisateur inconnu reçoit un message d'erreur clair

---

## 5. MUST 2 — Onboarding utilisateur via le chatbot

### Objectif

Un nouvel utilisateur doit pouvoir, en conversation naturelle, fournir ses centres d'intérêt, son email et son horaire préféré.

### Nœuds à utiliser

```
[AI Agent avec mémoire] → [Code: extraire les données] → [Supabase INSERT preferences]
```

### Configuration de l'AI Agent

**Modèle recommandé pour débuter :** OpenAI GPT-4o-mini (moins cher) ou Claude Haiku via l'API Anthropic.

**System prompt de l'agent :**

```
Tu es l'assistant de configuration de Persona, un service de newsletter personnalisée.

Ton rôle est de collecter ces informations auprès de l'utilisateur de façon conversationnelle :
1. Ses centres d'intérêt (minimum 1, maximum 10 topics)
2. Son adresse email
3. L'heure à laquelle il souhaite recevoir sa newsletter (format HH:MM)
4. Sa timezone (si il ne sait pas, propose UTC+1 pour la France)

Règles :
- Ne demande pas tout en même temps, guide l'utilisateur étape par étape
- Confirme chaque information avant de passer à la suivante
- Une fois toutes les infos collectées, résume-les et demande confirmation
- Quand l'utilisateur confirme, réponds UNIQUEMENT avec ce JSON (rien d'autre) :
  {"status": "complete", "email": "...", "topics": [...], "time": "HH:MM", "tz": "..."}
- Si l'utilisateur veut modifier une info, permets-le
- Rappelle toujours que l'utilisateur peut se désinscrire à tout moment

L'utilisateur est déjà authentifié. Session: {{$json.sessionId}}
```

**Activer la mémoire :** Dans l'AI Agent, configure **Memory** → "Window Buffer Memory" avec une fenêtre de 20 messages. Cela permet à l'agent de se souvenir du contexte de la conversation.

### Extraction des données collectées

Après l'AI Agent, ajoute un nœud **IF** qui vérifie si la réponse contient `"status": "complete"` :

```javascript
// Nœud Code — parser la réponse de l'IA
const response = $json.output;
try {
  // L'IA peut avoir mis du texte autour du JSON, on l'extrait
  const jsonMatch = response.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    const data = JSON.parse(jsonMatch[0]);
    if (data.status === 'complete') {
      return [{ json: { isComplete: true, ...data } }];
    }
  }
} catch(e) {}
return [{ json: { isComplete: false, message: response } }];
```

### Sauvegarde dans Supabase

Quand `isComplete: true` :
1. Nœud Supabase `INSERT` dans `preferences`
2. Nœud Supabase `UPDATE` dans `users` → mettre `gdpr_consent: true` et `gdpr_consent_date: now()`
3. Répondre à l'utilisateur : "Parfait ! Ta newsletter sera envoyée chaque jour à [heure]. À bientôt !"

---

## 6. MUST 3 — Récupération des news depuis plusieurs sources

### Objectif

Récupérer des articles d'actualité depuis au moins 2-3 sources différentes.

### Sources recommandées (APIs gratuites)

| Source | API | Limite gratuite | Nœud N8N |
|---|---|---|---|
| **NewsAPI** | newsapi.org | 100 req/jour | HTTP Request |
| **The Guardian** | open-platform.theguardian.com | 500 req/jour | HTTP Request |
| **RSS Feeds** | Tout site avec flux RSS | Illimité | RSS Feed Read |
| **GNews** | gnews.io | 100 req/jour | HTTP Request |

### Workflow de récupération (dans le Workflow B)

```
[Schedule Trigger]
        ↓
[Supabase: liste de tous les topics uniques de tous les users actifs]
        ↓
[Split In Batches] ← pour traiter topic par topic
        ↓
[HTTP Request: NewsAPI] + [RSS Feed Read: Guardian]
        ↓
[Merge] ← fusionner les résultats des deux sources
        ↓
[Code: déduplications + nettoyage]
        ↓
[Continuer vers le filtrage IA]
```

### Configuration du nœud HTTP Request pour NewsAPI

- **Method:** GET
- **URL:** `https://newsapi.org/v2/everything`
- **Query Parameters:**
  - `q` : `{{ $json.topic }}`
  - `language` : `fr`
  - `sortBy` : `publishedAt`
  - `pageSize` : `10`
  - `apiKey` : (mets ta clé dans les credentials ou directement ici pour commencer)

### Nœud RSS Feed

N8N a un nœud natif **RSS Feed Read**. Configure simplement l'URL du flux :
- Le Monde : `https://www.lemonde.fr/rss/une.xml`
- BBC : `https://feeds.bbci.co.uk/news/rss.xml`
- Reuters : `https://feeds.reuters.com/reuters/topNews`

### Nettoyage des données (nœud Code)

```javascript
// Normaliser les articles de toutes les sources
const articles = $input.all().map(item => {
  const a = item.json;
  return {
    json: {
      title: a.title || a.headline || 'Sans titre',
      description: a.description || a.summary || a.content?.substring(0, 300) || '',
      url: a.url || a.webUrl || a.link,
      source: a.source?.name || a.sectionName || 'Source inconnue',
      publishedAt: a.publishedAt || a.webPublicationDate || new Date().toISOString(),
      topic: $('Split In Batches').item.json.topic
    }
  };
});

// Déduplications par URL
const seen = new Set();
return articles.filter(a => {
  if (seen.has(a.json.url)) return false;
  seen.add(a.json.url);
  return true;
});
```

---

## 7. MUST 4 — Filtrage IA par utilisateur

### Objectif

Pour chaque utilisateur, l'IA sélectionne les articles pertinents parmi tous ceux récupérés, et génère un résumé court.

### Architecture

```
[Articles récupérés] + [Préférences utilisateur depuis Supabase]
                ↓
         [AI Agent: filtrage]
                ↓
         [Code: parser JSON de l'IA]
                ↓
         [Articles filtrés avec résumés]
```

### Prompt de filtrage

Dans un nœud **AI Agent** ou **Basic LLM Chain** :

```
Tu es un éditeur de newsletter. Voici les préférences d'un abonné :
- Centres d'intérêt : {{ $json.topics }}
- Langue souhaitée : {{ $json.language }}
- Nombre d'articles souhaités : {{ $json.num_articles }}

Voici une liste d'articles disponibles aujourd'hui :
{{ JSON.stringify($('Merge articles').all().map(i => i.json)) }}

Sélectionne exactement {{ $json.num_articles }} articles qui correspondent le mieux aux centres d'intérêt de cet abonné.
Pour chaque article sélectionné, génère un résumé de 2-3 phrases en {{ $json.language }}.

Réponds UNIQUEMENT avec ce JSON, sans texte avant ni après :
{
  "articles": [
    {
      "title": "...",
      "summary": "...",
      "url": "...",
      "source": "..."
    }
  ]
}
```

> ⚠️ Attention à la taille du contexte ! Si tu as beaucoup d'articles, tu peux dépasser la fenêtre de contexte du modèle. Limite à 20-30 articles en entrée.

### Optimisation des tokens (COULD)

Pour économiser des tokens, tu peux pré-filtrer par topic avant d'envoyer à l'IA :

```javascript
// Nœud Code — pré-filtre simple par mots-clés avant l'IA
const userTopics = $json.topics; // ex: ['tech', 'IA', 'climat']
const articles = $('Merge').all().map(i => i.json);

const filtered = articles.filter(a => {
  const text = (a.title + ' ' + a.description).toLowerCase();
  return userTopics.some(topic => text.includes(topic.toLowerCase()));
});

// Garde au max 20 articles pour l'IA
return filtered.slice(0, 20).map(a => ({ json: a }));
```

---

## 8. MUST 5 — Envoi de la newsletter par email

### Objectif

Envoyer un email bien formaté à l'utilisateur avec les articles sélectionnés.

### Options d'envoi d'email dans N8N

| Option | Avantage | Inconvénient |
|---|---|---|
| **Gmail** (nœud natif) | Simple, gratuit | 500 emails/jour max |
| **Send Email (SMTP)** | Fonctionne avec n'importe quel provider | Config SMTP nécessaire |
| **Brevo (ex-Sendinblue)** | 300 emails/jour gratuits, bon delivrability | Compte à créer |
| **Resend** | Simple, 100 emails/jour gratuits | API Key |

### Nœud Gmail — configuration

1. Dans N8N → Credentials → New → "Gmail OAuth2"
2. Suis le guide Google OAuth
3. Dans le nœud Gmail :
   - **To:** `{{ $json.email }}`
   - **Subject:** `Ta newsletter du {{ new Date().toLocaleDateString('fr-FR') }}`
   - **Email Type:** HTML (pour le COULD 1) ou Text
   - **Message:** voir ci-dessous

### Template email texte simple (pour commencer)

```javascript
// Nœud Code — construire le corps de l'email
const articles = $json.articles; // sortie de l'IA
const userName = $json.username;
const date = new Date().toLocaleDateString('fr-FR', { weekday: 'long', day: 'nuuser', month: 'long' });

let body = `Bonjour ${userName},\n\n`;
body += `Voici ta sélection d'actualités pour ${date} :\n\n`;
body += `${'─'.repeat(50)}\n\n`;

articles.forEach((article, i) => {
  body += `${i + 1}. ${article.title}\n`;
  body += `   Source : ${article.source}\n`;
  body += `   ${article.summary}\n`;
  body += `   → Lire : ${article.url}\n\n`;
});

body += `${'─'.repeat(50)}\n`;
body += `Pour modifier tes préférences ou te désinscrire, réponds à ce message ou écris à notre chatbot.\n`;
body += `\nPersona — Ta newsletter personnalisée par IA`;

return [{ json: { emailBody: body, emailSubject: `📰 Ta newsletter du ${date}` } }];
```

---

## 9. MUST 6 — Planification (schedule par utilisateur)

### Problème à résoudre

N8N ne permet pas un schedule différent par utilisateur nativement. La solution : un **Schedule Trigger qui tourne toutes les heures** et vérifie quels utilisateurs doivent recevoir leur newsletter à cet instant.

### Logique de vérification

```
[Schedule Trigger: toutes les heures]
        ↓
[Code: calculer l'heure actuelle en UTC]
        ↓
[Supabase: SELECT users + preferences WHERE schedule_time = heure_actuelle AND is_active = true]
        ↓
[IF: résultats trouvés?]
        ↓ Oui
[SplitInBatches: traiter 1 user à la fois]
        ↓
[Récupérer news + Filtrer + Envoyer]
```

### Nœud Code — vérification de l'heure

```javascript
// Calculer l'heure actuelle arrondie à l'heure
const now = new Date();
const currentHour = now.getUTCHours().toString().padStart(2, '0') + ':00:00';

return [{ json: { currentHour } }];
```

### Requête Supabase

Dans le nœud Supabase, utilise une requête personnalisée (mode "Execute Query") :

```sql
SELECT u.id, u.email, u.username, p.topics, p.language, p.num_articles, p.custom_prompt
FROM users u
JOIN preferences p ON p.user_id = u.id
WHERE p.schedule_time = '{{ $json.currentHour }}'
  AND u.is_active = true
  AND u.gdpr_consent = true;
```

> Pour les timezones : c'est un problème complexe. Pour commencer, demande aux utilisateurs leur heure en UTC. Pour la version avancée, utilise la bibliothèque `luxon` dans un nœud Code pour convertir.

---

## 10. MUST 7 — RGPD de base & désinscription

### Ce que tu dois implémenter minimum

**Consentement :** Lors de l'inscription via le chatbot, l'agent doit explicitement mentionner :
- Que l'email sera stocké et utilisé pour envoyer des newsletters
- Que l'utilisateur peut se désinscrire à tout moment
- Le consentement doit être enregistré dans la DB (`gdpr_consent = true`)

**Désinscription :** Deux méthodes à implémenter :

#### Méthode 1 — Via le chatbot

Dans ton AI Agent du Workflow A, gère l'intent "désinscription" :

```javascript
// Si l'IA détecte une intention de désinscription
// Supabase UPDATE users SET is_active = false WHERE id = userId
// Supabase UPDATE users SET email = 'DELETED_' + id WHERE id = userId (anonymisation)
```

#### Méthode 2 — Lien dans chaque email (Workflow C)

Ajoute en bas de chaque newsletter :

```
Pour vous désinscrire : https://ton-n8n.com/webhook/unsubscribe?uid={{ userId }}&token={{ token }}
```

Puis crée un **Workflow C** avec un **Webhook Trigger** :

```
[Webhook: GET /unsubscribe]
        ↓
[Code: vérifier uid + token]
        ↓
[Supabase: UPDATE is_active = false]
        ↓
[Respond to Webhook: "Vous avez bien été désinscrit."]
```

> **Sécurité du lien :** Le `token` doit être un hash unique par utilisateur (`sha256(userId + secret_salt)`) pour éviter qu'un attaquant désinscrive quelqu'un d'autre.

### Droit à l'oubli (suppression de données)

```javascript
// Option 1 : vraie suppression
// DELETE FROM users WHERE id = userId

// Option 2 : anonymisation (plus safe, garde les logs)
// UPDATE users SET email = 'deleted_' + id, username = 'deleted_' + id, is_active = false
// DELETE FROM preferences WHERE user_id = userId
```

---

## 11. COULD 1 — Newsletter en HTML

### Pourquoi c'est utile

Les emails HTML sont bien plus lisibles et professionnels. N8N + Gmail supportent parfaitement le HTML.

### Template HTML simple

```javascript
// Nœud Code — générer le HTML
const articles = $json.articles;
const userName = $json.username;
const date = new Date().toLocaleDateString('fr-FR', { weekday: 'long', day: 'nuuser', month: 'long' });

const articleCards = articles.map(a => `
  <div style="border-left: 3px solid #6366f1; padding-left: 16px; margin-bottom: 24px;">
    <h2 style="margin:0 0 4px; font-size:16px; color:#1e1e2e;">
      <a href="${a.url}" style="color:#6366f1; text-decoration:none;">${a.title}</a>
    </h2>
    <p style="margin:0 0 8px; font-size:12px; color:#888;">${a.source}</p>
    <p style="margin:0; font-size:14px; color:#333; line-height:1.6;">${a.summary}</p>
  </div>
`).join('');

const html = `
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"></head>
<body style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; background: #f9f9f9;">
  <div style="background:#6366f1; color:white; padding:24px; border-radius:8px 8px 0 0; text-align:center;">
    <h1 style="margin:0; font-size:24px;">📰 Persona</h1>
    <p style="margin:8px 0 0; opacity:0.8;">Ta newsletter du ${date}</p>
  </div>

  <div style="background:white; padding:24px; border-radius:0 0 8px 8px;">
    <p style="color:#555;">Bonjour <strong>${userName}</strong>, voici ta sélection du jour :</p>
    ${articleCards}
    <hr style="border:none; border-top:1px solid #eee; margin:24px 0;">
    <p style="font-size:12px; color:#aaa; text-align:center;">
      Pour te désinscrire : <a href="UNSUBSCRIBE_LINK" style="color:#6366f1;">cliquer ici</a>
    </p>
  </div>
</body>
</html>
`;

return [{ json: { html } }];
```

Dans le nœud Gmail, sélectionne **"HTML"** comme type de message et utilise `{{ $json.html }}`.

---

## 12. COULD 2 — Commentaires sur les newsletters passées

### Concept

L'utilisateur peut écrire dans le chatbot : "La newsletter d'hier était bien, mais je n'aime pas les articles de tech pure, préfère de l'IA appliquée". L'IA mémorise ces préférences pour les prochaines newsletters.

### Implémentation

**Dans le Workflow A (chatbot) :**

```javascript
// L'AI Agent détecte les feedbacks et les stocke
// System prompt à ajouter :

"Si l'utilisateur donne du feedback sur sa newsletter ou ses préférences,
sauvegarde ce feedback avec le JSON suivant :
{\"action\": \"save_feedback\", \"feedback\": \"[texte du feedback]\"}
Ce feedback sera utilisé pour personnaliser les prochaines newsletters."
```

Quand l'agent retourne `action: save_feedback` :
- Nœud Supabase : `UPDATE newsletter_logs SET feedback = '...' WHERE user_id = x ORDER BY sent_at DESC LIMIT 1`
- Ou créer une table `user_notes` avec les préférences accumulées

**Dans le Workflow B (envoi newsletter) :**

Récupérer les derniers feedbacks de l'utilisateur et les injecter dans le prompt de filtrage :

```javascript
// Ajouter au prompt de l'IA :
`Feedbacks des newsletters précédentes de cet utilisateur :
${previousFeedbacks.join('\n')}
Tiens compte de ces feedbacks pour ta sélection.`
```

---

## 13. COULD 3 — Fact-checking

### Concept simple

Avant d'inclure un article, vérifier qu'il est mentionné par au moins une autre source.

### Implémentation légère

```javascript
// Nœud Code — vérification croisée simple
// Pour chaque article, chercher si le même sujet est couvert par une autre source

const articles = $input.all().map(i => i.json);
const checkedArticles = [];

for (const article of articles) {
  // Cherche des articles avec des mots-clés similaires
  const keywords = article.title.split(' ')
    .filter(w => w.length > 4)
    .slice(0, 3);

  const confirmedBy = articles.filter(a =>
    a.url !== article.url &&
    a.source !== article.source &&
    keywords.some(kw => a.title.toLowerCase().includes(kw.toLowerCase()))
  );

  checkedArticles.push({
    ...article,
    factCheckScore: confirmedBy.length, // 0 = non confirmé, 1+ = confirmé
    confirmedBySources: confirmedBy.map(a => a.source)
  });
}

// Prioriser les articles confirmés
return checkedArticles
  .sort((a, b) => b.factCheckScore - a.factCheckScore)
  .map(a => ({ json: a }));
```

### Version avancée avec IA

Utilise un second appel LLM pour demander à l'IA d'évaluer la cohérence d'une information avec d'autres sources récupérées.

---

## 14. COULD 4 — Prompt personnalisé par utilisateur

### Concept

Chaque utilisateur peut customiser le ton, le format, la longueur de sa newsletter via le chatbot.

### Dans le chatbot (Workflow A)

Ajoute à l'AI Agent :

```
Si l'utilisateur veut personnaliser sa newsletter (ton, format, longueur, style),
récupère sa demande et réponds avec :
{"action": "update_prompt", "custom_prompt": "..."}

Exemples de prompts personnalisés :
- "Rédige comme si tu t'adressais à un expert en IA, avec du vocabulaire technique"
- "Sois bref, 3 articles max, style télégraphique"  
- "Utilise un ton humoristique et léger"
```

### Dans le Workflow B

Utilise le `custom_prompt` de la table `preferences` dans ton appel LLM :

```javascript
const systemPrompt = user.custom_prompt
  ? `${basePrompt}\n\nInstructions personnalisées pour cet utilisateur : ${user.custom_prompt}`
  : basePrompt;
```

---

## 15. Ordre de construction recommandé

Voici l'ordre optimal pour construire le projet en évitant les blocages :

```
SEMAINE 1 — Base fonctionnelle
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

☐ Étape 1 : Créer les tables Supabase (section 3)
☐ Étape 2 : Workflow B partiel — récupérer des news et les afficher (section 6)
            → Juste pour tester que tes APIs fonctionnent
☐ Étape 3 : Workflow A — Chat Trigger + AI Agent simple qui répond (section 4-5)
            → Sans authentification pour l'instant
☐ Étape 4 : Ajouter l'onboarding dans le Workflow A (section 5)
☐ Étape 5 : Compléter le Workflow B — filtrage IA + envoi email (sections 7-8)

SEMAINE 2 — Solidification
━━━━━━━━━━━━━━━━━━━━━━━━━━

☐ Étape 6 : Ajouter l'authentification (section 4)
☐ Étape 7 : Planification par utilisateur (section 9)
☐ Étape 8 : RGPD + désinscription, Workflow C (section 10)

SEMAINE 3 — Bonus
━━━━━━━━━━━━━━━━━

☐ Étape 9 : Newsletter HTML (section 11) ← priorité haute pour la démo
☐ Étape 10 : Feedback utilisateur (section 12)
☐ Étape 11 : Fact-checking (section 13)
☐ Étape 12 : Prompt personnalisé (section 14)
```

---

## 16. Erreurs fréquentes des débutants N8N

### Expressions mal formées

```
❌ {{ json.email }}        → manque le $
✅ {{ $json.email }}

❌ {{ $json['email'] }}    → fonctionne aussi mais moins lisible
✅ {{ $json.email }}

❌ {{ $node['Mon Node'].json.email }}   → ancienne syntaxe
✅ {{ $('Mon Node').item.json.email }}  → nouvelle syntaxe
```

### Nœud qui ne reçoit rien

Si un nœud semble ne pas recevoir de données, c'est souvent parce que le nœud précédent retourne un tableau vide. Ajoute un nœud **IF** ou **Code** pour déboguer :

```javascript
// Nœud Code de debug
console.log('Données reçues:', JSON.stringify($input.all()));
return $input.all(); // passe-plat
```

### AI Agent vs Basic LLM Chain

- **AI Agent** : a de la mémoire, peut utiliser des outils (ex: rechercher dans Supabase), multi-tours. Utilise pour le chatbot.
- **Basic LLM Chain** : appel unique sans mémoire. Utilise pour le filtrage/résumé dans le Workflow B.

### Credentials dans le JSON exporté

Avant de partager ton fichier JSON :
1. N8N > Settings > Export
2. Ouvre le JSON
3. Cherche les champs `"credentials"` et remplace les valeurs par des placeholders
4. Cherche les nœuds HTTP Request pour supprimer les headers d'auth

### Erreur "Workflow could not be started"

Vérifie que :
- Le workflow est bien **activé** (toggle en haut à droite)
- Le nœud trigger est bien configuré
- Les credentials sont valides (petit indicateur vert sur le nœud)

---

## 17. Checklist avant la démo

### Fonctionnalités MUST

- [ ] Le chatbot est accessible via l'URL générée par le Chat Trigger
- [ ] Un nouvel utilisateur peut s'inscrire
- [ ] Un utilisateur existant peut se connecter
- [ ] Un utilisateur connecté peut fournir ses préférences (topics, email, horaire)
- [ ] Le workflow récupère des news depuis au minimum 2 sources
- [ ] L'IA filtre les news selon les préférences de l'utilisateur
- [ ] Un email est envoyé à l'utilisateur avec les news sélectionnées
- [ ] La newsletter est envoyée à l'heure choisie par l'utilisateur
- [ ] Un utilisateur peut se désinscrire
- [ ] Les données personnelles sont protégées (un user A ne peut pas voir les données de user B)

### Sécurité

- [ ] Les mots de passe sont hachés (pas stockés en clair)
- [ ] Les sessions expirent
- [ ] Le lien de désinscription contient un token signé
- [ ] Le JSON exporté ne contient pas de credentials sensibles

### RGPD

- [ ] Le consentement est demandé et tracé en base
- [ ] La désinscription supprime ou anonymise les données
- [ ] L'email de newsletter contient un lien de désinscription

### Pour la présentation

- [ ] Préparer un compte de démo (ne pas utiliser ton vrai email)
- [ ] Avoir des news en cache ou s'assurer que les APIs fonctionnent le jour J
- [ ] Pouvoir expliquer pourquoi N8N vs coder from scratch (rapidité, no-code, intégrations natives, open-source)
- [ ] Montrer le workflow visuellement dans l8N (c'est un argument de vente fort)

---

*Guide rédigé pour le projet Persona — N8N v1.x | Supabase*
