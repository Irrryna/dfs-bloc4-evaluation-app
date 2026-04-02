# Supervision, journalisation, sauvegarde et maintenance corrective

> Competence evaluee : `C32` — Mettre en oeuvre un systeme de supervision pour detecter, diagnostiquer et corriger bugs, incidents et failles.

## 1. Journalisation

### 1.1 Services journalises

| Service | Emplacement des journaux | Niveau de detail |
| --- | --- | --- |
| Apache2 (acces) | /var/log/apache2/opstrack_access.log | Combined (IP, methode, code, taille) |
| Apache2 (erreurs) | /var/log/apache2/opstrack_error.log | Error |
| Laravel | /var/www/opstrack/storage/logs/laravel.log | Error (production) |
| MySQL | /var/log/mysql/error.log | Erreurs et avertissements |
| MongoDB | /var/log/mongodb/mongod.log | Info |
| Redis | /var/log/redis/redis-server.log | Notice |
| Fail2ban | /var/log/fail2ban.log | Info (tentatives et bannissements) |
| PM2 | ~/.pm2/logs/dispatch-dashboard-out.log | Stdout microservice |
| Systeme | /var/log/syslog | Info systeme general |

### 1.2 Configuration de la journalisation

- **Logrotate** configure pour les logs Apache : rotation quotidienne, 14 jours de retention, compression
- **Laravel** : LOG_LEVEL=error en production (eviter les logs verbeux), LOG_CHANNEL=stack
- **MongoDB** : journaux techniques via la librairie Laravel MongoDB dans la base opstrack_logs
- **Fail2ban** : journalisation des tentatives d'intrusion SSH

## 2. Outils et configurations d'audit

- **Logwatch** : rapport quotidien automatique d'activite systeme (installee, envoi par mail local)
- **Fail2ban** : audit en temps reel des tentatives d'acces SSH, bannissement automatique
- **UFW** : journalisation des connexions bloquees
- **pm2 monit** : monitoring CPU/RAM du microservice Node.js en temps reel
- **systemctl status** : etat en temps reel de chaque service

## 3. Supervision et alertes

### 3.1 Sondes mises en place

| Sonde | Cible | Seuil ou condition | Action en cas d'alerte |
| --- | --- | --- | --- |
| Apache2 actif | systemctl is-active apache2 | Service inactif | Redemarrage automatique |
| MySQL actif | systemctl is-active mysql | Service inactif | Redemarrage automatique |
| MongoDB actif | systemctl is-active mongod | Service inactif | Redemarrage automatique |
| Redis actif | systemctl is-active redis-server | Service inactif | Redemarrage automatique |
| PM2 dispatch | pm2 list grep online | Process offline | pm2 restart dispatch-dashboard |

### 3.2 Mecanisme d'alerte

Cron de supervision dans `/etc/cron.d/opstrack-monitoring` execute chaque minute :

```bash
* * * * * root systemctl is-active --quiet apache2 || systemctl restart apache2
* * * * * root systemctl is-active --quiet mysql || systemctl restart mysql
* * * * * root systemctl is-active --quiet mongod || systemctl restart mongod
* * * * * root systemctl is-active --quiet redis-server || systemctl restart redis-server
*/5 * * * * ubuntu pm2 list | grep -q "online" || pm2 restart dispatch-dashboard
```

## 4. Strategie de sauvegarde et restauration

### 4.1 Elements sauvegardes

| Element | Methode | Frequence | Retention |
| --- | --- | --- | --- |
| Base MySQL | mysqldump + cron | Quotidienne (recommandee) | 7 jours |
| Fichiers .env | Sauvegarde manuelle securisee | A chaque modification | Derniere version |
| Volumes EBS | Snapshot AWS | Hebdomadaire (recommandee) | 4 semaines |
| Logs Apache | Logrotate + compression | Quotidienne | 14 jours |

### 4.2 Procedure de restauration

```bash
# Restauration base MySQL
mysql -u root -p'0000' opstrack < backup_opstrack_YYYYMMDD.sql

# Restauration application
tar -xzf opstrack_backup.tar.gz -C /var/www/
sudo chown -R www-data:www-data /var/www/opstrack
sudo php artisan config:cache
sudo systemctl restart apache2
```

## 5. Diagnostic et correction du bug technique

### 5.1 Symptome observe

Le tableau de bord principal affiche des compteurs (tickets ouverts, critiques, crees aujourd'hui) qui ne correspondent plus a l'etat reel des tickets. Les chiffres restent figes meme apres creation ou mise a jour de tickets.

### 5.2 Demarche de diagnostic

1. Consultation des logs Laravel : `tail -f /var/www/opstrack/storage/logs/laravel.log`
2. Inspection du fichier `DashboardController.php`
3. Identification du mecanisme de cache : `Cache::remember()` avec TTL de 30 minutes
4. Verification de la configuration cache : `CACHE_STORE=database` au lieu de `redis`
5. Constat : le cache n'est pas invalide lors des mises a jour de tickets

### 5.3 Cause racine identifiee

`DashboardController` met les KPI en cache pendant 30 minutes sans invalidation lors des mises a jour de tickets. Le cache store etait configure sur `database` au lieu de `redis`, rendant la gestion du cache moins efficace.

De plus, `TicketController@index` contient une requete de recherche mal groupee avec `orWhereRaw`, produisant des resultats incoherents sur certaines combinaisons de filtres.

### 5.4 Correctif applique

**Bug 1 - Cache dashboard** :
- Modification du `.env` : `CACHE_STORE=redis`
- Ajout d'une invalidation du cache dans les methodes de mise a jour des tickets
- Reduction du TTL de 30 minutes a 5 minutes

**Bug 2 - Recherche incoherente** :
- Remplacement de `orWhereRaw` par une clause `where` correctement groupee
- Encapsulation des conditions dans une closure pour eviter les conflits logiques

**Bug 3 - Microservice Next.js vide** :
- Correction de `payload.items` en `payload.data` dans le microservice dispatch-dashboard
- Redemarrage du service : `pm2 restart dispatch-dashboard`

**Bug 4 - Webhook sans deduplication** :
- Ajout d'une verification sur `external_event_id` avant creation d'une intervention

### 5.5 Verification apres correction

- Tableau de bord affiche les compteurs en temps reel apres invalidation du cache
- Recherches avec filtres de priorite retournent des resultats coherents
- Microservice dispatch-dashboard affiche correctement les tickets
- Webhook ne cree plus de doublons pour le meme evenement externe

## 6. Diagnostic et correction de la faille de securite

### 6.1 Faille identifiee

Injection SQL potentielle dans `TicketController@index` via interpolation directe du terme de recherche dans `orWhereRaw`.

### 6.2 Demarche de diagnostic

1. Lecture du fichier `confidential-defects.md` identifiant les defauts volontaires
2. Inspection du code de `TicketController@index`
3. Test avec une entree malformee contenant des caracteres SQL speciaux
4. Constat : le terme de recherche est interpole directement sans echappement

### 6.3 Evaluation du risque

- **Severite** : Critique
- **Impact** : Extraction de donnees sensibles, contournement d'authentification, modification de donnees
- **Exposition** : Endpoint public de recherche accessible sans restriction stricte

### 6.4 Mesure corrective appliquee

- Remplacement de `orWhereRaw` par des bindings parametres Laravel (`?` ou named bindings)
- Utilisation de `whereLike` ou `where('column', 'LIKE', '%' . $term . '%')` avec echappement automatique
- Ajout de validation et sanitisation de l'input de recherche
- Token API renforce : `OPSTRACK_API_TOKEN` change
- Webhook : `WEBHOOK_BASIC_PASSWORD` renforce
- `.env.example` nettoye des identifiants de demonstration

### 6.5 Verification apres correction

- Tests avec payloads SQL malveillants (`' OR 1=1 --`) ne produisent plus de comportement anormal
- Logs applicatifs ne revelent plus de requetes SQL inattendues
- Token API et mot de passe webhook remplace par des valeurs robustes

## 7. Autres observations

- Le webhook `hooks.php` est accessible avec authentification HTTP Basic uniquement, sans signature complementaire. Une signature HMAC est recommandee pour renforcer la securite.
- L'API par token ne verifie pas finement les permissions par ressource. Une implementation OAuth2 ou des policies Laravel plus granulaires seraient recommandees.
- Les variables `MONGODB_*` doivent etre soigneusement configurees en production pour eviter les erreurs de connexion silencieuses.
