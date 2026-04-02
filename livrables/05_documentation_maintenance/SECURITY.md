# Journal de securite

Ce document recense les failles de securite identifiees pendant l'epreuve, leur evaluation et les mesures correctives appliquees.

## Faille 1

| Champ | Description |
| --- | --- |
| Date de detection | 2 avril 2026 |
| Composant concerne | `TicketController@index` |
| Description de la faille | Injection SQL via interpolation directe du terme de recherche dans `orWhereRaw` sans echappement ni binding parametre |
| Severite estimee | `Critique` |
| Impact potentiel | Extraction de donnees sensibles, contournement d'authentification, modification ou suppression de donnees en base |
| Mesure corrective appliquee | Remplacement de `orWhereRaw` par des bindings parametres Laravel. Validation et sanitisation de l'input de recherche. |
| Statut | `Corrige` |
| Preuve de correction | Tests avec payloads SQL malveillants ne produisent plus de comportement anormal |

## Faille 2

| Champ | Description |
| --- | --- |
| Date de detection | 2 avril 2026 |
| Composant concerne | `public/hooks.php` — webhook entrant |
| Description de la faille | Webhook protege uniquement par HTTP Basic Auth, sans signature complementaire ni restriction d'origine. Appele chaque minute, exposé publiquement. |
| Severite estimee | `Haute` |
| Impact potentiel | Usurpation d'identite de la source webhook, injection de donnees malveillantes dans l'application |
| Mesure corrective appliquee | Renforcement du mot de passe HTTP Basic. Recommandation d'ajout d'une signature HMAC-SHA256 pour les appels futurs. |
| Statut | `Mitigation en place` |
| Preuve de correction | Mot de passe webhook renforce dans .env (wh-SECURE-2026) |

## Faille 3

| Champ | Description |
| --- | --- |
| Date de detection | 2 avril 2026 |
| Composant concerne | API REST `/api/v1` — token d'authentification |
| Description de la faille | Token API avec valeur par defaut `change-me` en qualification. Verification des permissions par ressource insuffisante. |
| Severite estimee | `Haute` |
| Impact potentiel | Acces non autorise a l'API, lecture ou modification de donnees metier |
| Mesure corrective appliquee | Token API remplace par une valeur securisee en production. |
| Statut | `Corrige` |
| Preuve de correction | OPSTRACK_API_TOKEN=prod-token-2026-secure dans .env production |

## Faille 4

| Champ | Description |
| --- | --- |
| Date de detection | 2 avril 2026 |
| Composant concerne | `.env.example` |
| Description de la faille | Le fichier .env.example contenait les identifiants de demonstration utilises en qualification (mots de passe base de donnees, tokens) |
| Severite estimee | `Moyenne` |
| Impact potentiel | Exposition des identifiants par defaut facilitant les attaques sur les environnements non durcis |
| Mesure corrective appliquee | Nettoyage du .env.example, remplacement des valeurs reelles par des placeholders generiques |
| Statut | `Corrige` |
| Preuve de correction | .env.example ne contient plus de valeurs sensibles reelles |

## Faille 5

| Champ | Description |
| --- | --- |
| Date de detection | 2 avril 2026 |
| Composant concerne | Configuration Apache — `Options Indexes` |
| Description de la faille | Le VirtualHost Apache autorisait le listing des repertoires (Options Indexes FollowSymLinks) |
| Severite estimee | `Moyenne` |
| Impact potentiel | Exposition de la structure des fichiers de l'application aux visiteurs |
| Mesure corrective appliquee | Remplacement par `Options -Indexes +FollowSymLinks` dans la configuration VirtualHost |
| Statut | `Corrige` |
| Preuve de correction | Acces direct aux repertoires retourne une erreur 403 |
