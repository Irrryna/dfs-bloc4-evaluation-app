# Architecture cible et choix de l'hebergement

> Competence evaluee : `C29` — Selectionner une plateforme d'hebergement adaptee aux exigences techniques, economiques, qualitatives et reglementaires.

## 1. Analyse des besoins techniques

OpsTrack Field Service est une application de gestion d'interventions terrain reposant sur une stack multi-composants : Laravel 12, MySQL, MongoDB, Redis, Next.js et Apache2. Elle expose une API REST versionnee, un webhook appele chaque minute, et un microservice frontend. Ces besoins impliquent un hebergement offrant un controle total sur la configuration des services, une faible latence, et une capacite a evoluer.

## 2. Architecture cible proposee

### 2.1 Diagramme de deploiement

```
Internet
    |
[Route 53 DNS]
    |
[AWS EC2 - Qualification]          [AWS EC2 - Production]
 t3.small / eu-west-3               t3.medium / eu-west-3
 35.181.154.1                       51.44.17.160
 eval-dfs-q-tpl-20262-06            eval-dfs-p-tpl-20262-06
    |                                   |
 [Apache2 :80]                     [Apache2 :80/:443 + SSL]
    |                                   |
 [PHP 8.4 / Laravel 12]            [PHP 8.4 / Laravel 12]
 [MySQL 8.0]                       [MySQL 8.0]
 [MongoDB 8.0]                     [MongoDB 8.0]
 [Redis 7.0]                       [Redis 7.0]
 [Node.js / PM2 :3000]             [Node.js / PM2 :3000]
```

### 2.2 Description des composants

| Composant | Service ou technologie | Dimensionnement | Justification |
| --- | --- | --- | --- |
| Serveur web | Apache2 + mod_proxy | Inclus EC2 | Reverse proxy vers Next.js, gestion SSL |
| Application | PHP 8.4 / Laravel 12 | Inclus EC2 | Framework principal, API REST, webhook |
| BDD relationnelle | MySQL 8.0 | Inclus EC2 | Données métier transactionnelles |
| BDD NoSQL | MongoDB 8.0 | Inclus EC2 | Journaux techniques et événements |
| Cache | Redis 7.0 | Inclus EC2 | Cache applicatif, sessions |
| Microservice | Next.js / PM2 | Inclus EC2 | Dashboard dispatch, consomme API Laravel |
| DNS | CNAME it-students.fr | Route 53 | Résolution domaine vers IP EC2 |
| SSL | Let's Encrypt / Certbot | Gratuit | Certificat TLS automatique, renouvelé auto |

## 3. Choix du fournisseur et des services

### 3.1 Fournisseur retenu

**Amazon Web Services (AWS)** — Region eu-west-3 (Paris)
- Qualification : instance EC2 t3.small (i-0505cba1fd20959f9)
- Production : instance EC2 t3.medium (i-08c4cdc708317600b)

### 3.2 Justification du choix

AWS EC2 a ete retenu pour les raisons suivantes :

- **Controle total** : installation libre de toute la stack technique (PHP, MySQL, MongoDB, Redis, Node.js)
- **Elasticite native** : redimensionnement vertical sans migration, Auto Scaling possible
- **Geographie** : region Paris (eu-west-3) pour conformite RGPD et faible latence
- **Ecosysteme DevOps** : integration native GitHub Actions, CloudWatch, SNS
- **Disponibilite** : SLA 99,99% garanti contractuellement
- **Securite** : Security Groups, acces SSH par cle, isolation reseau

## 4. Estimation des couts

| Poste de depense | Cout mensuel estime | Cout annuel estime |
| --- | --- | --- |
| EC2 t3.small (qualification) | 15 € | 180 € |
| EC2 t3.medium (production) | 30 € | 360 € |
| EBS 20 Go gp3 x2 | 8 € | 96 € |
| Transfert données sortant | 5 € | 60 € |
| Route 53 DNS | 1 € | 12 € |
| **Total** | **59 €** | **708 €** |

Reduction possible de 30-40% via Reserved Instances (engagement 1 an).

## 5. Elasticite et evolutivite

- **Vertical** : changement de type d'instance EC2 en quelques minutes (stop/resize/start)
- **Horizontal** : ajout d'instances derriere un Elastic Load Balancer si la charge augmente
- **Stockage** : volumes EBS gp3 extensibles a chaud sans interruption
- **Cache** : migration vers ElastiCache possible si Redis devient un goulot d'etranglement
- **Base de donnees** : migration vers RDS possible pour haute disponibilite

## 6. Disponibilite et continuite de service

- SLA AWS EC2 : 99,99% de disponibilite
- Supervision cron : redemarrage automatique des services en cas de defaillance
- PM2 : redemarrage automatique du microservice Node.js
- Certbot : renouvellement automatique du certificat SSL
- Logrotate : rotation des logs pour eviter la saturation disque

## 7. Securite et sauvegarde

- Acces SSH exclusivement par cle privee ED25519
- UFW : pare-feu applicatif limitant les ports exposes (22, 80, 443)
- Fail2ban : protection contre les attaques brute-force SSH
- APP_DEBUG=false en production
- Secrets dans .env (exclu du depot Git)
- Sauvegardes : snapshots EBS recommandes (a automatiser)

## 8. Conformite et contraintes reglementaires

- Donnees hebergees en France (region eu-west-3 Paris) : conformite RGPD facilitee
- Certificat SSL Let's Encrypt : chiffrement des donnees en transit
- Acces administrateur trace via logs SSH et journaux systeme
- Aucune donnee personnelle stockee hors de l'UE
