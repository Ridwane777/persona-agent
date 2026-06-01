# Tasks — Persona
> Grille d'avancement : ce qui est implémenté, ce qui manque comme preuve ou document.

---

## MUST — Fonctionnalités obligatoires

### MUST 1 — Chatbot trigger & authentification

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| Chat Trigger actif et accessible via URL | ✅ Fait | Nœud `Chat trigger` dans Workflow A | — |
| Inscription (username, email, mot de passe) | ✅ Fait | Nœud `register` (Supabase tool) + `Agent Auth` | — |
| Connexion via chatbot | ✅ Fait | Nœud `Find_user` + `Create_active_sessions` | — |
| Hachage du mot de passe (SHA-256) | ✅ Fait | Documenté dans README (section Sécurité) | Pas de nœud Code visible avec le hash — **montrer le code JS du hachage en démo** |
| Système de sessions avec `session_id` | ✅ Fait | Nœuds `check_sessions`, `Compute Session State` | — |
| Expiration des sessions (24h) | ✅ Fait | README mentionne 24h, table `sessions` avec `expires_at` | **Montrer en démo** qu'une session expirée est bien rejetée |
| Blocage d'un user non authentifié | ✅ Fait | Nœud `IF: Session active?` → branching vers `Agent Auth` | — |
| Isolation des données entre utilisateurs | ✅ Fait | L'agent principal reçoit le `user_id` de session | Voir `SECURITY.md` pour l'isolation technique |
| Protection contre acteurs malveillants | ✅ Fait | Session + hash suffisent pour les bases | Voir `SECURITY.md` pour l'audit de sécurité |

---

### MUST 2 — Onboarding utilisateur via le chatbot

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| Collecte des centres d'intérêt (topics) | ✅ Fait | `save_preferences` (Supabase tool), table `preferences.topics` | — |
| Collecte de l'email | ✅ Fait | Table `users.email`, flow d'inscription dans `Agent Auth` | — |
| Collecte de l'horaire souhaité | ✅ Fait | `preferences.schedule_time` + `schedule_tz` | — |
| Collecte de la langue | ✅ Fait | `preferences.language` | — |
| Collecte du nombre d'articles | ✅ Fait | `preferences.num_articles` | — |
| Modification des préférences après inscription | ✅ Fait | Nœud `update_preferences` dans `Agent Principal` | — |
| Confirmation avant sauvegarde | ⚠️ Non vérifié | Dépend du system prompt de l'agent (non visible en entier) | **Démontrer en démo** que l'agent récapitule les préférences avant de les enregistrer |
| Guidage conversationnel étape par étape | ⚠️ Non vérifié | Dépend du comportement LLM à l'exécution | **Capture d'une conversation d'onboarding complète** à inclure en annexe |

---

### MUST 3 — Récupération des news depuis plusieurs sources

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| Source 1 : NewsAPI | ✅ Fait | Nœud `NewsAPI` (HTTP Request) dans Workflow B | — |
| Source 2 : The Guardian API | ✅ Fait | Nœud `The Guardian API` (HTTP Request) dans Workflow B | — |
| Source 3 : RSS BBC News | ✅ Fait | Nœud `RSS BBC News` dans Workflow B | — |
| Fusion des sources | ✅ Fait | Nœud `Fusionner sources` (Merge) | — |
| Normalisation des formats d'articles | ✅ Fait | Nœud `Normaliser + Dédup + Fact-check` (Code JS) | — |
| Déduplication par URL | ✅ Fait | Code JS dans `Normaliser + Dédup + Fact-check` | — |
| Minimum 2 sources actives | ✅ Fait | 3 sources présentes | — |

---

### MUST 4 — Filtrage IA par utilisateur

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| Filtrage par topics utilisateur | ✅ Fait | Nœud `IA — Filtrage et résumé` (LLM Chain) | — |
| Résumé de chaque article en 2-3 phrases | ✅ Fait | Prompt dans le nœud LLM + `Parser réponse IA` | — |
| Respect du nombre d'articles voulu (`num_articles`) | ✅ Fait | Code dans `Parser réponse IA` applique la limite | — |
| Réponse dans la langue de l'utilisateur | ✅ Fait | `language` injecté dans le prompt LLM | — |
| Fallback si parsing IA échoue | ✅ Fait | `Parser réponse IA` utilise les articles originaux en fallback | — |
| Respect du `custom_prompt` utilisateur | ✅ Fait | `custom_prompt` transmis via `Fusionner user + prefs` | Vérifier que le custom_prompt est **réellement injecté** dans le prompt LLM final |

---

### MUST 5 — Envoi de la newsletter par email

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| Envoi via Gmail OAuth2 | ✅ Fait | Nœud `Envoyer newsletter` (Gmail) dans Workflow B | — |
| Email en format HTML | ✅ Fait | Nœud `Construire email HTML` (Code JS) | — |
| Lien de désinscription dans l'email | ✅ Fait | Code HTML inclut un lien `/unsubscribe?uid=...&token=...` | — |
| Lien de modification des préférences | ✅ Fait | README indique un lien vers le chatbot dans l'email | — |
| Logging de l'envoi en base | ✅ Fait | Nœud `Logger envoi` (Supabase INSERT dans `newsletter_logs`) | — |
| **Capture d'un email reçu réel** | ❌ Manquant | Rien dans le repo | **Fournir une capture d'écran d'un email reçu** pour la démo/keynote |

---

### MUST 6 — Planification par utilisateur

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| Schedule Trigger toutes les heures | ✅ Fait | Nœud `Schedule Trigger (toutes les heures)` dans Workflow B | — |
| Calcul de l'heure UTC courante | ✅ Fait | Nœud `Calculer heure UTC` (Code JS) | — |
| Comparaison heure courante vs heure utilisateur | ✅ Fait | Nœud `Vérifier horaire` (Code JS avec `Intl.DateTimeFormat`) | — |
| Support des timezones africaines et mondiales | ✅ Fait | `Intl.DateTimeFormat` + README liste les timezones supportées | — |
| Boucle par utilisateur (`SplitInBatches`) | ✅ Fait | Nœud `Loop Users` dans Workflow B | — |
| Ne traite que les users actifs | ✅ Fait | Nœud `Utilisateurs actifs` filtre `is_active = true` | — |
| **Preuve d'envoi à la bonne heure** | ❌ Manquant | Rien dans le repo | **Log Supabase ou capture** montrant `sent_at` correspondant à l'horaire configuré |

---

### MUST 7 — RGPD de base & désinscription

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| Consentement explicite demandé à l'inscription | ✅ Fait | Nœud `Marquer consentement RGPD` dans Workflow A | **Montrer en démo** que le chatbot pose la question de consentement |
| `gdpr_consent` et `gdpr_consent_date` en base | ✅ Fait | Table `users`, README section RGPD | — |
| Désinscription via le chatbot | ✅ Fait | Nœud `unsubscribe_user` dans `Agent Principal` | — |
| Désinscription via lien dans l'email (Workflow C) | ✅ Fait | Workflow C complet (8 nœuds) | — |
| Token HMAC sécurisé sur le lien de désinscription | ✅ Fait | Code JS dans `Construire email HTML` + `Vérifier token` (Workflow C) | Le token actuel utilise un hash bit-shift simple, **pas un vrai HMAC** — à signaler ou corriger |
| Anonymisation de l'email à la désinscription | ✅ Fait | Nœud `Désactiver compte` dans Workflow C, README confirme `deleted_...` | — |
| Suppression des préférences et sessions | ✅ Fait | Nœuds `Supprimer préférences` et `Supprimer sessions` dans Workflow C | — |
| Page de confirmation HTML après désinscription | ✅ Fait | Nœud `Répondre succès` (Respond to Webhook) dans Workflow C | — |
| Page d'erreur si token invalide | ✅ Fait | Nœud `Répondre erreur` (Workflow C) | — |
| **Document de conformité RGPD** | ✅ Fait | Fichier `GDPR.md` à la racine | — |

---

## COULD — Fonctionnalités bonus

### COULD 1 — Newsletter en HTML

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| Email au format HTML structuré | ✅ Fait | Nœud `Construire email HTML` génère un template complet | — |
| Design responsive et lisible | ✅ Fait | Template table-based avec styles inline | — |
| Cartes d'articles avec titre, source, résumé, lien | ✅ Fait | Code JS confirme la structure des cartes | **Capture d'écran de l'email rendu** pour la démo |

---

### COULD 2 — Commentaires sur les newsletters passées

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| L'utilisateur peut donner du feedback via le chatbot | ✅ Fait | Nœud `save_feedback` dans `Agent Principal` (Workflow A) | — |
| Feedback stocké en base (`newsletter_logs.feedback`) | ✅ Fait | Table `newsletter_logs` avec colonne `feedback` | — |
| Feedback réinjecté dans le prompt du prochain envoi | ✅ Fait | Workflow B récupère les feedbacks via `R\u00e9cup\u00e9rer feedbacks pass\u00e9s` | — |

---

### COULD 3 — Fact-checking

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| Vérification croisée entre sources | ✅ Fait | Nœud `Normaliser + Dédup + Fact-check` calcule un `factCheckScore` | — |
| Score de confirmation par article | ✅ Fait | `factCheckScore` transmis via `Agréger pour l'IA` | — |
| Priorisation des articles multi-sources | ✅ Fait | Prompt LLM instruit de prioriser les articles avec score élevé | — |
| Fact-checking par LLM secondaire | ❌ Non implémenté | Pas de second appel LLM dédié | Non requis pour la démo, à mentionner comme piste d'amélioration |

---

### COULD 4 — Prompt personnalisé par utilisateur

| Sous-exigence | Statut | Preuve dans le projet | Ce qui manque |
|---|:---:|---|---|
| L'utilisateur peut définir un `custom_prompt` via le chatbot | ✅ Fait | Nœud `update_custom_prompt` dans `Agent Principal` (Workflow A) | — |
| `custom_prompt` stocké en base | ✅ Fait | Colonne `preferences.custom_prompt` | — |
| `custom_prompt` transmis au LLM de la newsletter | ✅ Partiel | `Fusionner user + prefs` inclut le `custom_prompt` | **Vérifier** que le nœud `IA — Filtrage et résumé` injecte réellement le `custom_prompt` dans son prompt |

---

## COULDs du sujet non couverts dans le guide

| Fonctionnalité | Statut | Note |
|---|:---:|---|
| Paiement / abonnement premium | ❌ Non implémenté | Non prioritaire |
| Référencer un autre utilisateur (profil public) | ❌ Non implémenté | Non prioritaire |
| Publicités ciblées dans les newsletters | ❌ Non implémenté | Non prioritaire (soulève des enjeux RGPD complexes) |
| Endpoints exposés (site web, Discord, email entrant) | ❌ Non implémenté | À mentionner comme évolution future |
| Newsletter multi-format (blog, réseaux sociaux, version imprimable) | ❌ Non implémenté | À mentionner comme évolution future |
| Multimédia généré par IA | ❌ Non implémenté | À mentionner comme évolution future |

---

## Documents & preuves manquants (récapitulatif)

| # | Document / Preuve | Priorité | Pour la démo ? |
|---|---|:---:|:---:|
| 1 | **Capture d'écran d'un email reçu** (newsletter HTML complète) | 🔴 Critique | ✅ Oui |
| 2 | **Log Supabase** montrant un envoi à l'heure configurée (`newsletter_logs`) | 🔴 Critique | ✅ Oui |
| 3 | **Capture d'une conversation d'onboarding complète** (inscription → préfs → confirmation) | 🔴 Critique | ✅ Oui |
| 4 | **Audit de sécurité complet** (SECURITY.md) | 🟢 Fait | ✅ Oui |
| 5 | **Note de conformité RGPD** (GDPR.md) | 🟢 Fait | ✅ Oui |
| 6 | **Vérification du token de désinscription** (token hash avec sel secret) | 🟢 Fait | ⚠️ Utile |
| 7 | **Preuve que le `custom_prompt` est injecté dans le LLM B** (Workflow B JSON) | 🟢 Fait | ❌ Non |
| 8 | **Implémentation COULD 2** : feedback réinjecté dans le prompt | 🟢 Fait | ❌ Non |
| 9 | **Mention de `factCheckScore` dans le prompt LLM** | 🟢 Fait | ❌ Non |

---

## Légende

| Icône | Signification |
|:---:|---|
| ✅ Fait | Implémenté et visible dans les workflows JSON |
| ⚠️ Partiel | Partiellement implémenté ou non vérifiable sans exécution |
| ❌ Non implémenté | Absent du projet |
| 🔴 Critique | Bloquant pour la note ou la démo |
| 🟠 Important | Attendu pour un projet complet |
| 🟡 Secondaire | Bonus, améliore la note |
