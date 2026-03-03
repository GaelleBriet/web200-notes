# WEB200 Notes

Cheatsheets de préparation OSWA

## 🚀 Setup local (prévisualisation)

```bash
# 1. Installer mkdocs-material
pip install mkdocs-material

# 2. Prévisualiser en local
mkdocs serve
# → http://127.0.0.1:8000
```

## 📤 Déploiement GitHub Pages

### Premier déploiement (manuel)

```bash
# 1. Créer repo GitHub : web200-notes (PUBLIC)
# 2. Cloner et pousser
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/TONPSEUDO/web200-notes.git
git push -u origin main

# 3. Déployer
mkdocs gh-deploy
```

Le site sera disponible à : `https://TONPSEUDO.github.io/web200-notes/`

### Déploiements suivants (automatiques)

Grâce au GitHub Action dans `.github/workflows/deploy.yml`, chaque `git push` sur `main` redéploie automatiquement le site.

```bash
# Workflow quotidien
git add .
git commit -m "Add XSS cheatsheet"
git push
# → GitHub Actions déploie automatiquement
```

## 📁 Structure

```
docs/
├── index.md               # Homepage
├── outils/                # Nmap, curl, gobuster, wfuzz, ffuf...
├── vulns/                 # SQLi, XSS, SSTI, XXE, LFI...
│   └── sql/               # SQL enumeration & payloads
├── post-exploit/          # Linux post-RCE, reverse shells
├── methodology/           # Workflows, checklist exam, payloads
├── setup/                 # Config VM, VPN
└── stylesheets/           # CSS custom
```

## ✏️ Ajouter une fiche

1. Créer un fichier `.md` dans le bon dossier
2. L'ajouter dans `mkdocs.yml` sous `nav:`
3. `mkdocs serve` pour prévisualiser
4. `git add . && git commit && git push`

## 📝 Syntaxe utile

```markdown
!!! tip "Conseil"
    Texte du conseil

!!! warning "Attention"
    Texte d'attention

!!! danger "Critique"
    Information critique

=== "MySQL"
    Code MySQL ici

=== "PostgreSQL"
    Code PostgreSQL ici
```
