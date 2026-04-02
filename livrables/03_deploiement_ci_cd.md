# Deploiement automatise

> Competence evaluee : `C31` — Mettre en oeuvre un systeme de deploiement automatise respectant les bonnes pratiques DevOps.

## 1. Strategie de deploiement

### 1.1 Vue d'ensemble

Strategie de deploiement continu (CD) basee sur GitHub Actions, declenchee automatiquement a chaque push sur la branche `main`. Le pipeline couvre les environnements de qualification et de production.

Deux niveaux de deploiement :
- **Script shell** : `/usr/local/bin/deploy-opstrack.sh` pour deploiement manuel ou automatise
- **GitHub Actions** : `.github/workflows/deploy.yml` pour deploiement continu via CI/CD

### 1.2 Diagramme du pipeline

```
Push sur main
      |
      v
[GitHub Actions]
      |
      +---> Checkout du code
      |
      +---> Connexion SSH a la production
      |
      +---> composer install (sans dev)
      |
      +---> php artisan config:cache
      +---> php artisan route:cache
      +---> php artisan migrate --force
      |
      +---> systemctl reload apache2
      +---> pm2 restart dispatch-dashboard
      |
      v
[Production mise a jour]
```

## 2. Outillage retenu

| Outil | Role dans le pipeline | Justification |
| --- | --- | --- |
| GitHub Actions | Orchestration CI/CD | Integration native avec le depot de code |
| appleboy/ssh-action | Connexion SSH et execution des commandes | Action eprouvee pour le deploiement SSH |
| Composer | Gestion des dependances PHP | Gestionnaire officiel pour Laravel |
| PHP Artisan | CLI Laravel (migrations, cache) | Outil natif de Laravel |
| PM2 | Gestion du microservice Node.js | Gestionnaire de processus robuste |
| Bash script | Deploiement manuel reproductible | Fallback en cas de probleme CI/CD |

## 3. Declenchement du deploiement

### 3.1 Mode de declenchement

- **Automatique** : sur chaque push vers la branche `main`
- **Manuel** : execution du script `/usr/local/bin/deploy-opstrack.sh` sur le serveur

```yaml
on:
  push:
    branches: [main]
```

### 3.2 Reproductibilite

Le deploiement peut etre relance a l'identique de deux facons :
1. Via GitHub Actions : re-executer le workflow depuis l'interface GitHub
2. Via le script shell : `sudo /usr/local/bin/deploy-opstrack.sh`

Les deux methodes produisent le meme resultat car elles executent les memes etapes.

## 4. Controles prealables au deploiement

| Controle | Description | Critere de passage |
| --- | --- | --- |
| Connexion SSH | Verification de l'acces SSH au serveur de production | Connexion etablie sans erreur |
| Composer install | Installation des dependances sans erreur | Exit code 0 |
| Migration Laravel | Application des migrations en attente | Aucune erreur de migration |
| Apache config | Validation de la configuration Apache | apache2ctl configtest OK |

## 5. Mise a jour de la production

Etapes effectives executees sur le serveur de production :

```bash
cd /var/www/opstrack
composer install --no-interaction --no-dev --optimize-autoloader
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan migrate --force
systemctl reload apache2
pm2 restart dispatch-dashboard
```

## 6. Verification post-deploiement

### 6.1 Smoke tests

| Test | Commande ou methode | Resultat attendu |
| --- | --- | --- |
| Application accessible | curl -I https://eval-dfs-p-tpl-20262-06.it-students.fr | HTTP 200 |
| Apache actif | systemctl is-active apache2 | active |
| MySQL actif | systemctl is-active mysql | active |
| Redis actif | systemctl is-active redis-server | active |
| MongoDB actif | systemctl is-active mongod | active |
| PM2 dispatch | pm2 list | dispatch-dashboard online |

### 6.2 Preuve de deploiement reussi

Application OpsTrack accessible et fonctionnelle sur :
- https://eval-dfs-p-tpl-20262-06.it-students.fr (production avec SSL)
- http://eval-dfs-q-tpl-20262-06.it-students.fr (qualification)

Tableau de bord affichant les tickets INC-240301 et INC-240302 correctement.

## 7. Conduite a tenir en cas d'echec

1. **Diagnostic** : consulter les logs GitHub Actions et les logs Apache (`/var/log/apache2/opstrack_error.log`)
2. **Rollback** : revenir au commit precedent via `git revert` ou `git checkout <commit>`
3. **Redemarrage services** : `sudo systemctl restart apache2 mysql mongod redis-server`
4. **Notification** : informer l'equipe via le canal de communication habituel

## 8. Scripts et fichiers de configuration

| Fichier | Role |
| --- | --- |
| `.github/workflows/deploy.yml` | Pipeline GitHub Actions pour deploiement automatique |
| `/usr/local/bin/deploy-opstrack.sh` | Script de deploiement manuel reproductible |
| `/etc/apache2/sites-available/opstrack.conf` | Configuration VirtualHost HTTP |
| `/etc/apache2/sites-available/opstrack-le-ssl.conf` | Configuration VirtualHost HTTPS (Let's Encrypt) |
| `/var/www/opstrack/.env` | Variables d'environnement de production |
| `/etc/cron.d/opstrack-monitoring` | Cron de supervision et redemarrage automatique |
