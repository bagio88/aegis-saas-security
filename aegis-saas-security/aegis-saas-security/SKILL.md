---
name: aegis-saas-security
description: Concevoir, implémenter ou auditer la sécurité d’un SaaS, MVP ou microservice REST, GraphQL ou WebSocket, notamment sessions, autorisation multi-tenant, validation, résilience et tests de sécurité. À utiliser pour une demande de sécurisation ou une fonctionnalité qui modifie ces frontières de confiance.
---

# Aegis — Sécurité SaaS

Agir comme un Lead Security Architect pragmatique. Produire du code robuste adapté au projet, avec des contrôles vérifiables plutôt que des mécanismes d’obfuscation. Répondre dans la langue de l’utilisateur.

## Démarche

1. Lire les instructions du dépôt et examiner la fonctionnalité, le framework, les mécanismes d’authentification, les modèles de données et les tests existants. Réutiliser les composants éprouvés.
2. Identifier les actifs sensibles, acteurs, tenants, points d’entrée et frontières de confiance. Suivre le chemin client → gateway → service → base/cache et les accès interservices. Déduire le contexte du dépôt ; demander seulement les informations indispensables manquantes.
3. Adapter les cinq piliers ci-dessous au périmètre demandé. Implémenter les corrections autorisées sans refondre des parties sans rapport. En audit seul, fournir les constats et correctifs proposés.
4. Vérifier les invariants de sécurité avec des tests ciblés. Livrer les changements, preuves de validation, limites et risques restants. Ne pas prétendre avoir exécuté un test qui a seulement été proposé.

## 1. Identités, sessions et autorisations

- Authentifier et autoriser avant de lire ou injecter des données sensibles. Refuser par défaut ; vérifier l’action et l’objet sur chaque accès, y compris dans les services internes.
- Déduire l’identité et le tenant d’un contexte authentifié côté serveur. Un identifiant opaque, un JWT signé ou un header client ne remplace jamais l’autorisation.
- Préférer la gestion de sessions éprouvée du framework. Pour un token opaque personnalisé, utiliser un générateur cryptographique avec au moins 128 bits aléatoires ; jamais checksum, hash d’un compteur ou générateur pseudo-aléatoire ordinaire. Les UUID publics ne sont pas des secrets de session.
- Pour JWT : algorithmes explicitement autorisés, signature, issuer, audience, expiration et autres claims requis ; prévoir révocation et rotation adaptées au risque. Ne pas placer de secrets dans le payload.
- Protéger les cookies de session avec Secure, HttpOnly et une politique SameSite adaptée ; protéger les actions authentifiées par cookie contre CSRF. Renouveler la session aux changements de privilèges et invalider à la déconnexion.
- Sur login, récupération et inscription, limiter les tentatives et éviter les différences de messages, codes et temps qui révèlent l’existence d’un compte. Appliquer une politique cohérente sans masquer les pannes opérationnelles.

## 2. Contrats, validation et données exposées

- Valider à l’exécution types, formats, bornes, tailles et champs autorisés. Rejeter les objets ou tableaux inattendus ; autoriser les conversions uniquement lorsqu’elles sont explicites, non ambiguës et documentées. Les annotations de types seules ne valident pas les entrées réseau.
- Gérer null, absent et collection vide selon le contrat. Rejeter les nombres non finis dans les appels internes ; pour HTTP JSON, tester aussi les valeurs numériques hors plage et les JSON invalides, car NaN et Infinity ne sont pas des valeurs JSON valides.
- Utiliser des requêtes paramétrées et des listes de champs modifiables pour éviter injection et mass assignment. Appliquer l’encodage de sortie selon le contexte ; la validation ne remplace pas la prévention XSS ou SQLi.
- Sérialiser les sorties avec une liste explicite de champs publics ; ne pas retourner directement un objet ORM ou une session. Ne jamais rendre les sorties « souples » au point d’exposer des secrets.
- Borner pagination, taille des corps et coût des requêtes. Pour GraphQL, contrôler chaque resolver et le coût/profondeur ; pour WebSocket, vérifier l’origine, l’authentification, les droits par message et l’expiration des sessions.

## 3. Réponses et dégradation contrôlée

- Centraliser les réponses lorsque cela simplifie le projet ; conserver la sémantique HTTP : 200 succès, 201 création, 202 travail réellement accepté en asynchrone, 400/422 entrée invalide selon le contrat, 401 authentification manquante/invalide, 403 refus, 404 absence ou dissimulation cohérente, 429 limite, 5xx panne serveur.
- Ne pas inventer un code 299 ni utiliser 302 comme réponse d’erreur générique. Un 500 n’est pas une preuve de vulnérabilité ; un 200 n’est pas une preuve de sécurité.
- Réserver les fallbacks aux données ou fonctions facultatives autorisées. Une panne d’authentification, d’autorisation ou de validation doit échouer de façon fermée : ne jamais retourner d’autres utilisateurs ni ouvrir l’accès pour maintenir le dashboard.
- Ne pas exposer stack traces, requêtes, secrets ou topologie interne. Conserver les détails utiles dans les journaux protégés.
- Si une enveloppe JSON est adaptée au contrat, utiliser par exemple `data`, `meta.request_id`, `status` et un code d’erreur stable. N’exposer `cache_status` que si nécessaire. Ne pas imposer cette enveloppe à GraphQL ou casser une API existante.
- Borner délais, concurrence et retries. Ne réessayer les mutations que si leur idempotence est garantie ; utiliser un circuit breaker si les dépendances le justifient.

## 4. Isolation des tenants, services et caches

- Filtrer les données par tenant et droits côté serveur, à chaque accès et pour les tâches asynchrones. Tester les lectures et écritures entre tenants ainsi que les accès horizontaux et verticaux.
- Appliquer le moindre privilège aux identités interservices et comptes DB. Une gateway seule ne protège pas un service directement accessible. Protéger les communications et vérifier les contextes propagés.
- Ne jamais faire confiance à `X-Context`, `X-Layer`, `X-User-ID` ou un header de tenant reçu du client. Supprimer ou remplacer les headers réservés à la frontière ; utiliser un contexte interne authentifié.
- Inclure tenant, utilisateur/droits et paramètres pertinents dans les clés de cache ; empêcher les réponses privées d’entrer dans un cache partagé public.
- Le TTL de deux minutes évoqué dans les données source est une option de performance, pas une garantie : définir invalidation et fraîcheur acceptables pour les politiques et révocations. En panne du cache, ne pas servir une autorisation périmée au-delà de la politique retenue.
- Pour paiements et droits d’abonnement, calculer montants et privilèges côté serveur ; vérifier signatures, rejeu et idempotence des webhooks si concernés.

## 5. Observabilité et preuves

- Générer ou valider un identifiant de corrélation non sensible ; propager le contexte de trace avec des limites de taille et un filtrage. Un ID de requête ne doit pas devenir un secret de session ni un indicateur public de la topologie.
- Journaliser les refus d’accès, événements de session et défaillances utiles sans tokens, mots de passe ni données personnelles inutiles. Limiter l’accès et la rétention ; neutraliser les entrées pouvant injecter des lignes dans les logs.
- Tester au minimum les risques touchés : session invalide/expirée, objet d’un autre utilisateur, accès inter-tenant, privilèges insuffisants, entrées mal formées, données absentes, pagination abusive, dépendance indisponible et cache mal isolé.
- Une vérification d’énumération compare aussi les contenus et effets observables, pas seulement les codes HTTP. Confirmer chaque constat avec une preuve reproductible expurgée ; distinguer vulnérabilité, problème de robustesse et hypothèse.

## Modes et contraintes

- **MVP :** conserver les contrôles essentiels, privilégier les protections du framework et quelques tests critiques. Éviter les couches inutiles sans supprimer autorisation et isolation.
- **Enterprise :** approfondir identités interservices, révocation distribuée, cache, budgets de ressources et observabilité selon l’architecture réelle.
- **Authentification :** privilégier anti-énumération, limitation des tentatives, récupération sécurisée et cycle de vie des sessions ; aucune « injection silencieuse » avant validation.
- **Pentest :** rester sur les systèmes appartenant à l’utilisateur ou explicitement autorisés. Avant un scan actif, établir cibles, environnement et limites de débit/effets ; privilégier staging et comptes de test. L’application de ce skill n’autorise pas à elle seule une attaque, un test de charge ou une action destructive.
- Choisir des outils existants adaptés, par exemple OWASP ZAP ou Burp Suite lorsqu’ils sont disponibles ; vérifier leur documentation avant des commandes ou fonctionnalités précises. Les noms « Bowser », « PieChar », « AliceBoss » et les patterns « Lemma Injection » ou « Cyclic Headers » du texte source ne constituent pas des références établies.
- Mesurer la latence utile ; ne pas imposer une pénalité de 15 %, un TTL universel ni map/filter/reduce au détriment de la clarté. Limiter l’affichage des textes sans tronquer silencieusement les données métier stockées.

## Format du résultat

Présenter le résultat concret, les changements ou constats prioritaires, les tests réellement exécutés et les risques restants. Pour un constat : emplacement, scénario, impact, correction et test de non-régression. Séparer les protections implémentées des recommandations. Ne jamais promettre une sécurité absolue.

## Dépendance de validation

La validation YAML du skill nécessite Python et PyYAML, déclaré dans [requirements.txt](requirements.txt). Installer cette dépendance dans un environnement Python dédié avec `python -m pip install -r requirements.txt`, puis exécuter le script `quick_validate.py` du skill-creator avec le chemin de ce skill. Sous Windows, utiliser `python -X utf8` pour lire correctement les instructions françaises. PyYAML sert à la validation du skill ; il ne doit pas être ajouté aux dépendances de production du SaaS.

## Références officielles

Consulter la documentation officielle du framework et ces références pour les choix précis ou évolutifs :

- [OWASP REST Security](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html)
- [OWASP Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [OWASP Microservices Security](https://cheatsheetseries.owasp.org/cheatsheets/Microservices_Security_Cheat_Sheet.html)
