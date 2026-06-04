# Note de Conformité RGPD — Projet Persona

## 1. Introduction
Le projet **Persona** est un service d'agrégation d'actualités par IA. Dans le cadre de son fonctionnement, il traite des données à caractère personnel (DCP) appartenant à ses utilisateurs. Cette note détaille les mesures techniques et organisationnelles mises en place pour assurer la conformité au Règlement Général sur la Protection des Données (RGPD).

## 2. Données collectées
Le système collecte et stocke les données suivantes :
- **Username** : Pour l'identification.
- **Email** : Nécessaire pour l'envoi de la newsletter.
- **Mot de passe** : Stocké exclusivement sous forme de hash (SHA-256).
- **Centres d'intérêt (Topics)** : Pour la personnalisation.
- **Horaires et zone géographique** : Pour la planification des envois.
- **Feedback** : Commentaires facultatifs sur les newsletters reçues (via le chatbot ou les boutons "J'ai aimé" / "Je n'ai pas aimé" dans l'email). Ces feedbacks sont utilisés uniquement pour améliorer les prochains envois de l'utilisateur concerné.
- **Prompt personnalisé (custom_prompt)** : Instructions de style optionnelles définies par l'utilisateur pour adapter le ton de sa newsletter.

## 3. Base légale du traitement
Le traitement repose sur le **consentement explicite** de l'utilisateur (Article 6.1.a du RGPD). Ce consentement est recueilli lors de l'onboarding via le chatbot et enregistré en base de données (`gdpr_consent` = true) avec horodatage (`gdpr_consent_date`).

## 4. Durée de conservation
Les données sont conservées tant que l'utilisateur est actif. 
- En cas de désinscription, les données de préférences et les sessions sont **immédiatement supprimées**.
- L'adresse email est **anonymisée** (ex: `deleted_123@persona.local`) afin de conserver l'intégrité des logs d'envoi tout en supprimant le lien avec une personne physique identifiée.

## 5. Droits des utilisateurs
Les utilisateurs peuvent exercer leurs droits directement via le chatbot ou les liens dans les emails :
- **Droit à l'oubli / Effacement** : Accessible via le lien "Se désinscrire" présent dans chaque newsletter ou via la commande "Désinscription" dans le chatbot.
- **Droit d'accès et de rectification** : L'utilisateur peut consulter et modifier ses préférences à tout moment via le chatbot.
- **Droit à la portabilité** : Sur demande, les données stockées peuvent être extraites via les outils d'administration Supabase.

## 6. Sécurité des données
- **Hachage** : Les mots de passe ne sont jamais stockés en clair.
- **Isolation** : Chaque requête à la base de données est filtrée par l'ID de l'utilisateur de la session active, garantissant qu'un utilisateur ne peut pas accéder aux données d'un autre.
- **Tokens sécurisés** : Les liens de désinscription utilisent un token unique généré par un sel secret (`PERSONA_SALT`), empêchant toute tentative de désinscription malveillante par tiers.

## 7. Transfert de données
Les données sont stockées sur **Supabase** (infrastructure AWS/GCP selon configuration). Les résumés de news sont générés via **OpenRouter** (OpenAI/Anthropic). Aucune donnée personnelle (PII) autre que le prénom/username n'est transmise aux LLM lors du résumé des news (seuls les articles et les topics sont envoyés).

---
*Date de dernière mise à jour : 4 Juin 2026*