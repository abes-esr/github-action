# 🔧 Centralisation des Workflows GitHub Actions (abes-esr/github-action)

Ce dépôt centralise les workflows GitHub Actions réutilisables utilisés par les différents projets de l'**Abes**.

L'objectif est d'homogénéiser nos pipelines d'intégration et de déploiement continus (CI/CD) tout en simplifiant leur maintenance.

---

## 💡 Intérêts de la Centralisation

1. **DRY (Don't Repeat Yourself) / Réutilisabilité** : Évite de dupliquer des centaines de lignes de code YAML de configuration CI/CD dans chaque dépôt applicatif.
2. **Maintenance Unifiée et Simplifiée** :
   - Une correction de bug, une amélioration de sécurité ou une mise à jour de version d'action tierce (ex: monter d' `actions/checkout@v5` à `@v6`) est effectuée **à un seul endroit** (ici).
   - Tous les projets consommateurs en bénéficient immédiatement sans aucune intervention manuelle.
3. **Standardisation et Conformité** : Assure que tous les projets de l'organisation respectent les mêmes standards de qualité, de nommage, de sécurité (ex : restrictions des releases aux branches `main`/`master`, validation SemVer des tags) et de structure.
4. **Lisibilité des Dépôts Applicatifs** : Les fichiers de workflow dans les projets se limitent à de simples déclencheurs (callers) de quelques lignes, rendant la configuration CI/CD de chaque application extrêmement lisible.

---

## 🛠️ Workflows Disponibles

Chaque workflow est conçu pour être réutilisable (`on: workflow_call`).

- **`buildx-pubtodockerhub.yml`** : Compile et publie une image Docker multi-plateforme sur DockerHub avec Docker Buildx.
- **`java-create-release.yml`** : Calcule les versions (SemVer), met à jour les `pom.xml` Maven et le `README.md`, crée les tags Git, effectue les commits/pushes et publie la release officielle GitHub pour un projet Java.
- **`node-create-release.yml`** : Similaire à la release Java, mais adaptée pour les projets Node.js (met à jour `package.json` et le `README.md`).

---

## 🚀 Exemples d'Utilisation (Triggers)

Voici comment appeler ces workflows depuis vos dépôts applicatifs. Les fichiers déclencheurs doivent être placés dans le dossier `.github/workflows/` de votre application.

### 1. Exemple : Compilation et Publication Docker (`buildx-pubtodockerhub.yml`)

Créez un fichier `.github/workflows/docker-publish.yml` dans votre dépôt projet :

```yaml
name: "Build & Publish on DockerHub"

on:
  push:
    branches:
      - main
      - develop
    paths-ignore:
      - "**.md"
      - ".github/**"
  workflow_dispatch:

jobs:
  build-and-push:
    # Appel du workflow centralisé (utiliser @main ou une version tagguée spécifique)
    uses: abes-esr/github-action/.github/workflows/buildx-pubtodockerhub.yml@main
    with:
      dockerhub_image_prefix: "abesesr/theses"
      docker_tag_target: "api-recherche" # Cible multi-stage dans le Dockerfile & suffixe du tag de l'image
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```

⚠️ Lors de l'utilisation de ces workflows réutilisables dans un nouveau dépôt projet, veillez à bien vérifier et adapter les points suivants :

- **Nom du Stage dans le `Dockerfile` (`AS <stage_name>`)** :
  - Si vous utilisez le multi-stage build, assurez-vous que la directive `FROM ... AS <stage_name>` de votre `Dockerfile` correspond exactement à la valeur passée dans le paramètre `docker_tag_target` (ex : `api-recherche`, `batch-dump`). Le nom du stage sera aussi utilisé pour construire le tag de l'image finale (ex: `abesesr/theses:main-api-recherche`, `abesesr/theses:develop-batch-dump`).
- **Image DockerHub (`dockerhub_image_prefix`)** :
  - Vérifiez que le nom de l'image spécifié dans `dockerhub_image_prefix` correspond bien au dépôt cible de l'image souhaité sur DockerHub (ex: `abesesr/theses`).
- **Secrets de connexion** :
  - Assurez-vous que les secrets `DOCKERHUB_USERNAME` et `DOCKERHUB_TOKEN` sont bien configurés dans les secrets de votre dépôt applicatif.

### 2. Exemple : Release pour un projet Java Maven (`java-create-release.yml`)

Créez un fichier `.github/workflows/release.yml` dans votre dépôt projet :

```yaml
name: "Create Release"

on:
  workflow_dispatch:
    inputs:
      versionType:
        description: "Quel type de version ? (major, minor, patch)"
        required: true
        default: "patch"
        type: choice
        options:
          - major
          - minor
          - patch

jobs:
  release:
    uses: abes-esr/github-action/.github/workflows/java-create-release.yml@main
    with:
      versionType: ${{ github.event.inputs.versionType }}
    secrets:
      TOKEN_GITHUB_FOR_GITHUB_ACTION: ${{ secrets.TOKEN_GITHUB_FOR_GITHUB_ACTION }}
```

⚠️ Lors de l'utilisation de ces workflows réutilisables dans un nouveau dépôt projet, veillez à bien vérifier et adapter les points suivants :

- **Droits du Token GitHub (`TOKEN_GITHUB_FOR_GITHUB_ACTION`)** :
  - Le token passé en secret doit disposer des privilèges d'écriture (`contents: write`) sur le dépôt pour pouvoir pousser les commits de version, créer et pousser le tag git de la release, et créer la release GitHub.
- **Fichiers de configuration et de version** :
  - **Java** : Assurez-vous que le projet contient bien un fichier `pom.xml` valide à la racine (et dans ses sous-modules si applicables).
  - **Node.js** : Assurez-vous que le projet contient un fichier `package.json` valide.
  - **README.md** : Le fichier `README.md` de votre projet doit contenir une ligne contenant le motif `Version: X.Y.Z` (ex: `Version: 1.0.0`) si vous souhaitez qu'elle soit mise à jour automatiquement lors de la release.

---

## 📌 Bonnes Pratiques

- **Versioning des workflows** : Lorsque vous référencez un workflow centralisé, préférez pointer vers une branche stable (ex: `@main`) pour éviter les régressions inattendues lors de modifications du dépôt central.
- **Secrets** : Les secrets (comme `DOCKERHUB_TOKEN` ou `TOKEN_GITHUB_FOR_GITHUB_ACTION`) peuvent être partagés à l'organisation afin de réutiliser plus facilement ces workflows.
