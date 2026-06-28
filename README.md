# IAM Reference Site

Base de connaissances personnelle — IAM, OAuth2, Entra ID.

## Démarrage rapide

### 1. Installer les dépendances

```bash
pip install mkdocs-material
```

### 2. Lancer en local

```bash
mkdocs serve
# → http://127.0.0.1:8000
```

### 3. Modifier le contenu

Chaque section est un fichier Markdown dans `docs/`.  
Édite, sauvegarde — le site se recharge automatiquement.

### 4. Déployer sur GitHub Pages

```bash
# Option A — automatique (recommandée)
git add . && git commit -m "update" && git push
# → GitHub Actions se charge du build + deploy

# Option B — manuel
mkdocs gh-deploy
```

## Ajouter une nouvelle page

1. Créer le fichier `docs/section/ma-page.md`
2. L'ajouter dans `mkdocs.yml` sous `nav:`
3. `git push` → déploiement automatique

## Structure

```
docs/
├── index.md                    ← Page d'accueil
├── fondamentaux/               ← Définitions, architecture
├── oauth2/                     ← Protocoles, flows, tokens
├── entra-id/                   ← CA, Token Protection, CAE, PIM
├── implementation/             ← Patterns MSAL par type d'app
├── monitoring/                 ← Logs, KQL, Sentinel
├── gouvernance/                ← Entitlement, Access Reviews
└── roadmap/                    ← Breaking Changes 2026
```
