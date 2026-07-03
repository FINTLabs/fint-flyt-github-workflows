# fint-flyt-github-workflows

Reusable GitHub workflows for FLYT repositories.

## Java app

```yaml
name: CI Build

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  build:
    uses: FINTLabs/fint-flyt-github-workflows/.github/workflows/java-app-build-image.yaml@refactor/reusable-github-workflows-pilot
    permissions:
      contents: read
      packages: write
    with:
      push-image: ${{ github.event_name == 'push' }}
    secrets: inherit
```

CD jobs call `java-app-cd-deploy.yaml` with a repository-specific JSON list of `org`/`cluster` targets. Manual deploys can call `java-app-deploy-aks.yaml` directly with one `org`, one `cluster` and the image reference produced by the build workflow.

## Java library

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  build:
    uses: FINTLabs/fint-flyt-github-workflows/.github/workflows/java-library-ci.yaml@refactor/reusable-github-workflows-pilot
```

Publishing to Reposilite is handled by `java-library-publish-reposilite.yaml` on release events.
