# Base de connaissances — Note de passation

Ce document est destine a un pair charge de reprendre la maintenance de l'application. Il doit permettre de comprendre le fonctionnement, les points d'attention et les procedures essentielles sans zone d'ombre majeure.

## 1. Presentation de l'application

OpsTrack Field Service est une application web de gestion d'interventions terrain. Elle permet aux equipes support et exploitation de suivre les tickets d'incidents, planifier les techniciens et superviser les interventions prioritaires. Elle est utilisee en interne par les equipes de maintenance et les dispatcheurs.

Fonctionnalites principales :
- Tableau de bord centralise des interventions (compteurs, liste des tickets)
- Gestion des tickets : creation, mise a jour, filtrage par priorite et statut
- API REST versionnee (/api/v1) pour integrations tierces
- Webhook entrant pour mises a jour automatiques depuis systemes externes
- Microservice dispatch-dashboard (Next.js) pour vue secondaire des interventions
- Enrichissement via API meteo Open-Meteo

## 2. Architecture technique

### 2.1 Composants principaux

| Composant | Technologie | Role |
| --- | --- | --- |
| Application principale | Laravel 12 / PHP 8.4 | Logique metier, API REST, webhook |
| Serveur web | Apache2 | Reverse proxy, SSL, serveur HTTP |
| Base relationnelle | MySQL 8.0 | Donnees metier transactionnelles |
| Base NoSQL | MongoDB 8.0 | Journaux techniques et evenements |
| Cache | Redis 7.0 | Cache applicatif et sessions |
| Microservice frontend | Next.js 18 / Node.js | Dashboard dispatch secondaire |
| Gestionnaire processus | PM2 | Maintien du microservice Node.js |
| API externe | Open-Meteo | Enrichissement donnees meteo |

### 2.2 Schema d'architecture

```
[Navigateur] --HTTPS--> [Apache2 :443]
                              |
                    +---------+---------+
                    |                   |
              [Laravel/PHP]    [/dispatch-dashboard]
                    |                   |
              [MySQL]          [Next.js :3000 via PM2]
              [MongoDB]
              [Redis]
                    |
              [Open-Meteo API]
```

## 3. Points d'attention connus

- **Cache dashboard** : les KPI sont mis en cache Redis. En cas de compteurs incorrects, vider le cache : `php artisan cache:clear`
- **Microservice Next.js** : surveiller avec `pm2 status`. Si hors ligne : `pm2 restart dispatch-dashboard`
- **Webhook hooks.php** : appele chaque minute par un cron externe. Verifier les logs Apache en cas de probleme
- **Extension MongoDB PHP** : version 2.2.1 installee manuellement. A verifier lors des mises a jour PHP
- **Certificat SSL** : expire le 2026-07-01, renouvellement automatique Certbot, verifier avec `certbot renew --dry-run`
- **Disque** : usage a 76% en production. Surveiller avec `df -h`, purger les logs anciens si necessaire

## 4. Procedures operationnelles

### 4.1 Deploiement

```bash
# Deploiement automatique via GitHub Actions (push sur main)
git push origin main

# Deploiement manuel
sudo /usr/local/bin/deploy-opstrack.sh
```

### 4.2 Sauvegarde et restauration

```bash
# Sauvegarde MySQL
mysqldump -u root -p'0000' opstrack > backup_$(date +%Y%m%d).sql

# Restauration
mysql -u root -p'0000' opstrack < backup_YYYYMMDD.sql

# Vider le cache apres restauration
php artisan cache:clear
php artisan config:cache
```

### 4.3 Supervision et alertes

```bash
# Etat des services
systemctl status apache2 mysql mongod redis-server

# Logs en temps reel
tail -f /var/log/apache2/opstrack_error.log
tail -f /var/www/opstrack/storage/logs/laravel.log
pm2 logs dispatch-dashboard

# Redemarrage d'urgence tous services
sudo systemctl restart apache2 mysql mongod redis-server
pm2 restart all
```

### 4.4 Acces et secrets

- SSH : `ssh -i ubuntu.pem ubuntu@51.44.17.160` (production) ou `ubuntu@35.181.154.1` (qualification)
- PhpMyAdmin : http://eval-dfs-q-tpl-20262-06.it-students.fr/phpmyadmin (qualification uniquement)
- Fichier .env : `/var/www/opstrack/.env` (ne pas commiter, ne pas partager)
- Secrets stockes uniquement dans .env sur le serveur

## 5. Bugs et failles corriges pendant l'epreuve

- **Cache KPI dashboard** : TTL reduit et invalidation ajoutee lors des mises a jour tickets
- **Recherche incoherente** : orWhereRaw remplace par bindings parametres securises
- **Microservice vide** : payload.items corrige en payload.data
- **Doublons webhook** : deduplication sur external_event_id ajoutee
- **Injection SQL** : TicketController@index securise contre les injections
- **APP_DEBUG** : desactive en production
- **Token API** : valeur par defaut remplacee

## 6. Ameliorations recommandees

- Automatiser les sauvegardes MySQL avec un cron quotidien
- Ajouter une signature HMAC-SHA256 sur le webhook hooks.php
- Implementer des permissions plus granulaires sur l'API REST
- Mettre en place des alertes email via AWS SNS ou un service de notification
- Configurer des snapshots EBS automatiques pour la sauvegarde des volumes
- Envisager une migration vers RDS pour la haute disponibilite MySQL

## 7. Contacts et ressources

- Depot de code : https://github.com/Irrryna/dfs-bloc4-evaluation-app
- Documentation Laravel : https://laravel.com/docs/12.x
- Documentation PM2 : https://pm2.keymetrics.io/docs/
- Let's Encrypt / Certbot : https://certbot.eff.org/
- API Open-Meteo : https://open-meteo.com/en/docs
