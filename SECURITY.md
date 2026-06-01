# Rapport d'Audit de Sécurité — Persona

## 1. Résumé Exécutif
L'architecture de Persona a été conçue avec une approche "Security by Design". Bien que le projet ne dispose pas d'interface graphique traditionnelle, l'interaction via chatbot et webhooks a été sécurisée pour prévenir les vecteurs d'attaque courants.

## 2. Authentification et Gestion des Sessions
- **Stockage des secrets** : Les mots de passe sont hashés en **SHA-256** côté workflow avant insertion dans Supabase. Aucune donnée en clair n'est stockée.
- **Cycle de vie des sessions** :
  - Chaque session est liée à un `session_id` unique.
  - Une vérification systématique de l'expiration (`expires_at`) est effectuée à chaque interaction.
  - Délai d'expiration automatique : 24 heures.
- **Protection contre le brute force** : L'utilisation d'outils LLM pour l'auth permet une flexibilité mais nécessite des garde-fous. L'Agent Auth est strictement limité par son system prompt à ne gérer que l'inscription/connexion.

## 3. Isolation des Données (Multi-ténacité)
- **Filtrage par ID** : Tous les nœuds Supabase utilisent des filtres stricts sur l'UUID de l'utilisateur (`user_id = {{ $json.user_id }}`). 
- **Séparation des flux** : Le Workflow B (Newsletter) boucle sur les utilisateurs et isole totalement le contexte de génération de chaque newsletter dans des exécutions séparées. Il est techniquement impossible qu'un utilisateur reçoive les news ou le custom prompt d'un autre.

## 4. Sécurité des Webhooks (Désinscription)
- **Tokenisation** : Le lien de désinscription `GET /unsubscribe?uid=...&token=...` est protégé par un token hashé.
- **Entropie** : Le token combine l'ID utilisateur avec un sel statique secret stocké dans le code du workflow. 
- **Validation** : Le Workflow C valide l'existence de l'utilisateur ET la validité du token avant toute action destructrice.

## 5. Protection contre les Injections
- **Prompt Injection** : Les entrées utilisateur (`custom_prompt`) sont traitées comme de la donnée brute et injectées dans un bloc de texte clairement délimité pour le LLM. L'utilisation de templates et de JSON pour les échanges IA réduit le risque de détournement de l'agent.
- **SQL Injection** : Les nœuds Supabase utilisent l'API PostgREST qui paramètre automatiquement les requêtes, éliminant les risques d'injection SQL directe.

## 6. Recommandations
1. **Rotation des clés** : Changer régulièrement le `PERSONA_SALT` utilisé pour les tokens de désinscription.
2. **Rate Limiting** : Implémenter un rate-limiting au niveau du Webhook N8N pour éviter les attaques par déni de service.
3. **Mise à jour HMAC** : (Optionnel) Utiliser le nœud "Crypto" natif de N8N pour un HMAC plus robuste si disponible dans l'environnement de déploiement.

---
*Audit réalisé le 1er Juin 2024*