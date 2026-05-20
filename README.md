# SPHERE-JOB

Dashboard de suivi de candidatures en cybersécurité, avec automatisation Gmail → IA → Google Sheets.

---

## Stack technique

| Composant | Technologie |
|-----------|-------------|
| Application | HTML/CSS/JS — hébergée sur GitHub Pages |
| Automatisation | n8n self-hosted (Docker) |
| Stockage | Google Sheets ("Sphere-job-2") |
| IA | Claude API (claude-sonnet-4-20250514) |
| Infrastructure | VM Kali Linux dans VirtualBox sur Windows |

---

## Architecture

```
      Gmail 
        ↓
   n8n (toutes les 15 min)
        ↓
   Code JS — extraction email
        ↓
   Claude API — classification + parsing
        ↓
   If node — "offre" ou "inconnu"
     ↓               ↓
Google Sheets     Google Sheets
 (onglet offres)  (onglet inconnus)
        ↓
   Mark as read
        ↓
   Sphere-Job App
  (lecture via Apps Script)
```

---

## Infrastructure

- VM Kali Linux dans VirtualBox sur Windows
- Docker container n8n avec `--restart unless-stopped`
- Redirection de port VirtualBox : 5678 → 5678
- Accès n8n : `http://localhost:5678`

## Installation

**Prérequis** : Docker installé sur la machine hôte.

```bash
# Démarrer n8n
docker run --name=n8n --restart unless-stopped \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -d docker.n8n.io/n8nio/n8n

# Mettre à jour
docker pull docker.n8n.io/n8nio/n8n && docker stop n8n && docker rm n8n
```

Accès : `http://localhost:5678`

---

## Workflow n8n

### Nœuds

1. **Schedule Trigger** — toutes les 15 minutes
2. **Agrégateur** (*******@gmail.com) — `getAll: message`, is:unread, Simplify OFF
3. **LinkedIn** (************@gmail.com) — `getAll: message`, is:unread, Simplify OFF
4. **Code JS** — extraction subject / from / date / heure / url / gmail_id + appel Claude API
5. **If** — `type === "offre"` → onglet offres / sinon → onglet inconnus
6. **Append row in sheet** (offres) — Google Sheets, onglet "offres"
7. **Append row in sheet** (inconnus) — Google Sheets, onglet "inconnus"
8. **Mark as read offre** — Gmail account 
9. **Mark as read inconnu** — Gmail account 

### Code JS (extraction + Claude)

Le nœud Code extrait les données brutes du mail (décodage base64, extraction URL) puis appelle Claude pour classifier l'email et extraire les champs structurés.

**Champs extraits par Claude :**
- `type` : "offre" ou "inconnu"
- `company` : nom de l'entreprise
- `role` : intitulé du poste
- `cat` : soc / pent / grc / oth
- `contract` : cdi / stage
- `salary` : fourchette salariale
- `location` : ville
- `exp` : expérience requise
- `score` : 0-10
- `desc` : résumé court

---

## Google Cloud / Credentials

| Service | Projet GCP | Compte |
|---------|------------|--------|
| Gmail OAuth2 | My Project 13917 Gmail | *********** |
| Gmail OAuth2 | My Project 13917 Gmail | *************** |
| Google Sheets OAuth2 | GoogleSheet-test | ***************** |



---

## Google Sheets

- **Spreadsheet ID** : `1jPMQy34vZ0pNPpyEl2Vr_GuIQCf_RYrIErWvFrvmtMM`
- **Onglet offres** : données des offres classifiées
- **Onglet inconnus** : emails non identifiés comme offres
- **Onglet historique** : offres sur lesquelles on a postulé

### Apps Script (API intermédiaire)

Un Google Apps Script expose les données du Sheet en JSON pour que l'application puisse les lire sans authentification côté client.

---

## Application Sphere-Job

Fichier : `sphere-job-final.html` — hébergé sur GitHub Pages.

### Fonctionnalités

- Dashboard temps réel des offres CDI et stages
- Scoring IA automatique à l'ajout d'une offre
- Classement par catégorie (SOC, Pentest, GRC, Autres)
- Statuts : Nouvelle / Postulée / À relancer / Rejetée
- Coups de cœur (liked)
- Assistant IA (analyse + email de relance)
- Détection des doublons via historique
- Gestion des CV (upload PDF)
- Badges de progression gamifiés
- Courrier inconnu (emails non classifiés)

### Configuration

> ⚠️ Note sécurité : la clé API est stockée en localStorage côté client. Usage personnel uniquement — ne pas utiliser en environnement partagé ou production.

---

