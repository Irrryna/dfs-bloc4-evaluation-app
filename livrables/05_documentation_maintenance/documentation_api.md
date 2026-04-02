# Documentation d'API

## 1. Vue d'ensemble de l'API

| Champ | Valeur |
| --- | --- |
| URL de base | https://eval-dfs-p-tpl-20262-06.it-students.fr/api/v1 |
| Format | `JSON` |
| Authentification | Token Bearer via header `Authorization: Bearer <token>` ou `X-Api-Token` |

## 2. Endpoints disponibles

| Methode | Endpoint | Description | Authentification requise |
| --- | --- | --- | --- |
| GET | /api/v1/tickets | Liste des tickets avec filtres optionnels | Oui |
| GET | /api/v1/tickets/{id} | Detail d'un ticket | Oui |
| POST | /api/v1/tickets | Creation d'un ticket | Oui |
| PUT | /api/v1/tickets/{id} | Mise a jour d'un ticket | Oui |
| DELETE | /api/v1/tickets/{id} | Suppression d'un ticket | Oui |
| GET | /api/v1/dashboard | KPI du tableau de bord | Oui |
| POST | /hooks.php | Webhook entrant (HTTP Basic Auth) | Oui (Basic) |

## 3. Exemples de requetes et reponses

### GET /api/v1/tickets

```bash
curl -H "Authorization: Bearer prod-token-2026-secure" \
  https://eval-dfs-p-tpl-20262-06.it-students.fr/api/v1/tickets
```

Reponse :
```json
{
  "data": [
    {
      "id": 1,
      "reference": "INC-240301",
      "title": "Intermittent payment terminal outage",
      "priority": "critical",
      "status": "in_progress",
      "technician": "Lina Perez",
      "site": "Lyon Confluence"
    }
  ],
  "meta": {
    "total": 2,
    "page": 1
  }
}
```

### POST /hooks.php

```bash
curl -u user:wh-SECURE-2026 \
  -H "Content-Type: application/json" \
  -d '{"external_event_id": "EVT-001", "status": "resolved", "ticket_ref": "INC-240301"}' \
  https://eval-dfs-p-tpl-20262-06.it-students.fr/hooks.php
```

## 4. Codes d'erreur

| Code | Signification |
| --- | --- |
| 200 | Succes |
| 201 | Ressource creee |
| 401 | Non authentifie (token manquant ou invalide) |
| 403 | Acces interdit (permissions insuffisantes) |
| 404 | Ressource non trouvee |
| 422 | Erreur de validation des donnees |
| 500 | Erreur serveur interne |
