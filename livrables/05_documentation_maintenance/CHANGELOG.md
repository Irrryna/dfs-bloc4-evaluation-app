# Changelog

Toutes les modifications notables apportees pendant l'epreuve sont documentees dans ce fichier.
Le format s'inspire de [Keep a Changelog](https://keepachangelog.com/).

## [Session du 2 avril 2026]

### Ajoute

- Configuration complete de l'environnement de production (Apache2, PHP 8.4, MySQL, MongoDB, Redis, Node.js)
- Certificat SSL Let's Encrypt sur eval-dfs-p-tpl-20262-06.it-students.fr
- Microservice Next.js dispatch-dashboard gere par PM2
- Pipeline CI/CD GitHub Actions (.github/workflows/deploy.yml)
- Script de deploiement manuel (/usr/local/bin/deploy-opstrack.sh)
- Pare-feu UFW avec regles strictes (ports 22, 80, 443)
- Protection Fail2ban contre les attaques SSH brute-force
- Cron de supervision et redemarrage automatique des services
- Logrotate pour la rotation des logs Apache
- Logwatch pour les rapports d'activite systeme

### Modifie

- `.env` production : APP_ENV=production, APP_DEBUG=false, LOG_LEVEL=error
- `.env` production : CACHE_STORE=redis (corrige depuis database)
- `.env` production : APP_URL mis a jour vers le domaine de production HTTPS
- `OPSTRACK_API_TOKEN` remplace par une valeur securisee
- `WEBHOOK_BASIC_PASSWORD` renforce

### Corrige

- Bug cache dashboard : KPI mis en cache 30 min sans invalidation -> TTL reduit et invalidation ajoutee
- Bug recherche tickets : orWhereRaw mal groupe -> remplace par bindings parametres
- Bug microservice Next.js : payload.items -> payload.data
- Bug webhook : creation de doublons sans deduplication sur external_event_id

### Securite

- Injection SQL dans TicketController@index via orWhereRaw -> remplace par requetes parametrees
- APP_DEBUG desactive en production (prevention d'exposition de stacktraces)
- Token API et mot de passe webhook renforces
- .env.example nettoye des identifiants de demonstration
- Acces SSH par cle uniquement, pas de mot de passe
