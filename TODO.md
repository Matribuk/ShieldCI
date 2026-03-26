# ShieldCI — TODO

> GitHub Action qui génère automatiquement des pipelines CI/CD DevSecOps complets
> et ouvre une PR avec les fichiers produits.
> **Stack : Go 1.22 · `google/go-github` · `text/template` · Docker**

---

## PHASE 0 — Setup du repo

- [ ] Créer le repo GitHub `ShieldCI` (public, MIT)
- [ ] Ajouter description : *"GitHub Action that auto-generates hardened CI/CD pipelines — lint, tests, Trivy, Gitleaks, SAST and more — and opens a PR with the generated workflows."*
- [ ] Topics : `github-actions`, `devops`, `devsecops`, `ci-cd`, `trivy`, `gitleaks`, `sast`, `pipeline`, `automation`, `security`, `golang`
- [x] Initialiser le module Go : `go mod init github.com/Richonn/shieldci`
- [x] Initialiser la structure de base :
  ```
  ShieldCI/
  ├── action.yml
  ├── Dockerfile
  ├── README.md
  ├── LICENSE (MIT)
  ├── DECISIONS.md
  ├── go.mod
  ├── go.sum
  ├── cmd/
  │   └── shieldci/
  │       └── main.go          ← entrypoint
  ├── internal/
  │   ├── detect/
  │   │   ├── detect.go
  │   │   └── detect_test.go
  │   ├── generate/
  │   │   ├── generate.go
  │   │   └── generate_test.go
  │   ├── pr/
  │   │   ├── pr.go
  │   │   └── pr_test.go
  │   └── config/
  │       └── config.go        ← parsing des inputs/env vars
  └── templates/
      ├── base/
      ├── node/
      ├── python/
      ├── java/
      ├── go/
      └── docker/
  ```
- [ ] Créer `DECISIONS.md` pour documenter les choix techniques (même pattern que KubeForge)

> ✅ **config.go** — parsing centralisé des env vars implémenté

---

## PHASE 1 — Définition de l'action (`action.yml`)

- [x] Définir les **inputs** :
  - `github-token` *(required)* — token pour créer la PR
  - `language` *(optional)* — override de détection : `node`, `python`, `java`, `go`, `auto`
  - `docker` *(optional, default: auto)* — forcer ou désactiver la détection Docker
  - `kubernetes` *(optional, default: false)* — inclure un job de déploiement K8s
  - `enable-trivy` *(optional, default: true)*
  - `enable-gitleaks` *(optional, default: true)*
  - `enable-sast` *(optional, default: true)* — CodeQL ou Semgrep
  - `sast-tool` *(optional, default: codeql)* — `codeql` | `semgrep`
  - `branch-name` *(optional, default: `shieldci/generated-workflows`)*
  - `pr-title` *(optional, default: `[ShieldCI] Add CI/CD DevSecOps pipeline`)*
- [x] Définir les **outputs** :
  - `pr-url` — URL de la PR créée
  - `detected-stack` — stack détecté (JSON)
  - `generated-files` — liste des fichiers générés
- [x] Configurer le runner : `using: docker`, `image: Dockerfile`

---

## PHASE 2 — Auto-détection du stack (`detect.go`)

- [x] Détecter le **langage principal** via présence de fichiers :
  - `package.json` → Node.js
  - `requirements.txt` / `pyproject.toml` / `setup.py` → Python
  - `pom.xml` / `build.gradle` → Java
  - `go.mod` → Go
  - `Cargo.toml` → Rust *(bonus)*
- [x] Détecter la présence d'un **Dockerfile** (racine ou sous-dossier)
- [x] Détecter la présence de **manifests K8s** (`k8s/`, `manifests/`, `helm/`, `Chart.yaml`)
- [x] Merger la détection avec les **overrides manuels** (inputs ont priorité)
- [x] Retourner un objet `StackConfig` structuré utilisé par `generate.go`
- [x] Couvrir tous les cas de détection avec `detect_test.go`

---

## PHASE 3 — Templates de workflows (`templates/`)

Chaque template est un fichier YAML `text/template` avec des variables injectées par `generate.go`.

### Template `base/security.yml.tmpl`
- [x] Job **Gitleaks** — détection de secrets dans l'historique git
- [x] Job **SAST CodeQL** — analyse statique multi-langage
- [x] Job **SAST Semgrep** — alternative open source (si `sast-tool: semgrep`)

### Template `node/ci.yml.tmpl`
- [x] Job lint (`eslint` / `prettier`)
- [x] Job tests (`npm test` / `jest`) avec matrix de versions Node
- [x] Job build

### Template `python/ci.yml.tmpl`
- [x] Job lint (`flake8` / `ruff`)
- [x] Job tests (`pytest`) avec matrix de versions Python
- [x] Job build

### Template `java/ci.yml.tmpl`
- [x] Job build + tests (`mvn` / `gradle`)
- [x] Job coverage (JaCoCo)

### Template `go/ci.yml.tmpl`
- [x] Job lint (`golangci-lint`)
- [x] Job tests (`go test`)

### Template `docker/build-scan.yml.tmpl`
- [x] Job build Docker image
- [x] Job **Trivy scan** de l'image buildée
- [x] Upload du rapport SARIF vers GitHub Security tab
- [x] Cache Docker layers entre runs

### Template `docker/k8s-deploy.yml.tmpl` *(si kubernetes: true)*
- [x] Job deploy via `kubectl` / Helm
- [x] Gestion des environments GitHub (staging / prod)

---

## PHASE 4 — Génération des fichiers (`generate.go`)

- [x] Charger les templates `text/template` selon le `StackConfig`
- [x] Injecter les variables (nom du repo, versions détectées, features activées)
- [x] Combiner les templates pertinents en fichiers `.yml` finaux
- [x] Écrire les fichiers dans `.github/workflows/` du repo cible
- [x] Produire un résumé lisible des fichiers générés (pour le body de la PR)
- [x] Couvrir la génération avec `generate_test.go`

---

## AMÉLIORATION — Reusable workflows générés

Refactoriser la génération pour produire des workflows séparés + un orchestrateur, au lieu d'un seul gros fichier par stack.

### Nouvelle structure générée
```
.github/workflows/
├── ci.yml              ← orchestrateur (uses: workflow_call vers les autres)
├── lint.yml            ← reusable
├── test.yml            ← reusable
├── docker.yml          ← reusable (si Docker détecté)
└── security.yml        ← reusable (Gitleaks + SAST)
```

- [x] Créer les nouveaux templates : `base/orchestrator.yml.tmpl`, `node/lint.yml.tmpl`, `node/test.yml.tmpl`, `python/lint.yml.tmpl`, `python/test.yml.tmpl`, `java/lint.yml.tmpl`, `java/test.yml.tmpl`, `go/lint.yml.tmpl`, `go/test.yml.tmpl`
- [x] Refactoriser `generate.go` pour générer un fichier par responsabilité
- [x] Mettre à jour `generate_test.go` pour les nouveaux comptes de fichiers
- [x] Mettre à jour les templates existants pour adopter `workflow_call`

---

## PHASE 5 — Création de la PR (`internal/pr/pr.go`)

- [x] Créer une branche dédiée (`shieldci/generated-workflows` par défaut)
- [x] Commiter les fichiers générés sur cette branche via GitHub API
- [x] Ouvrir une PR avec :
  - Titre configurable
  - Body auto-généré listant les fichiers créés + le stack détecté
  - Label `automated` / `ci-cd` (les créer si absents)
- [x] Gérer le cas où la branche/PR existe déjà (update plutôt que recréer)
- [x] Setter l'output `pr-url`

---

## PHASE 6 — Dockerfile de l'action

- [x] Base image légère : `golang:1.25-alpine AS builder`
- [x] Ajouter les dépendances : `go get github.com/google/go-github/v60`
- [x] Compiler le binaire Go statique : `CGO_ENABLED=0 GOOS=linux go build -o /shieldci ./cmd/shieldci`
- [x] Image finale `FROM alpine:3.19` — copier uniquement le binaire compilé
- [x] Entrypoint : `[\"/shieldci\"]`
- [x] Optimiser avec multi-stage build → image finale < 20MB
- [ ] Tester l'image localement avec `act` (outil de test GitHub Actions en local)

---

## PHASE 7 — Tests & CI de ShieldCI lui-même

- [x] Mettre en place `go test ./...` pour les tests unitaires (`go test -race ./...`)
- [x] Configurer un workflow CI pour ShieldCI (oui, le projet teste lui-même) :
  - lint Go (`golangci-lint`)
  - tests unitaires (`go test -race ./...`)
  - build de l'image Docker
  - Trivy scan de l'image
  - Gitleaks sur le repo
- [x] Viser une couverture > 80% sur `internal/detect` et `internal/generate` — detect: 91.7% / generate: 85.5%

---

## PHASE 8 — Documentation

- [x] **README.md** :
  - Badge CI
  - Pitch en 2 phrases
  - Quick start (snippet d'utilisation minimal)
  - Tableau de tous les inputs/outputs
  - Stack supportés
  - Roadmap
- [x] **DECISIONS.md** — documenter pourquoi `text/template`, pourquoi Docker action vs composite, etc.
- [ ] Ajouter des exemples dans `examples/` (repos fictifs avec les workflows générés)

---

## PHASE 9 — Publication sur le Marketplace

- [ ] Vérifier les prérequis Marketplace (description, icône, couleur dans `action.yml`)
- [ ] Ajouter `branding` dans `action.yml` :
  ```yaml
  branding:
    icon: zap
    color: orange
  ```
- [ ] Créer une release GitHub taguée (`v1.0.0`)
- [ ] Publier sur le GitHub Marketplace via l'interface repo
- [ ] Vérifier que l'action apparaît bien dans les résultats de recherche Marketplace

---

## BONUS (post-v1)

- [ ] Support Rust (`Cargo.toml`)
- [ ] Support monorepo (détecter plusieurs stacks dans le même repo)
- [ ] Output `detected-stack` exploitable pour d'autres actions en aval
- [ ] Mode `--dry-run` : affiche le YAML dans le Job Summary sans créer de PR
- [ ] Support Semgrep avec règles custom configurables
- [ ] Intégration SBOM (Software Bill of Materials) via Syft
