# fint-flyt-github-workflows

Felles gjenbrukbare GitHub Actions-workflows for FLYT-repoer.

Målet er at hvert applikasjons- og bibliotek-repo bare skal definere det som er
repo-spesifikt, for eksempel deploy-targets, mens selve bygging, publisering og
deploy ligger her.

## Java app

Applikasjonsrepoer kan bruke `java-app-build-image.yaml` for CI-bygg og
Docker-image. Pull requests bygger image lokalt i GitHub Actions uten å pushe,
mens push til `main` bygger og pusher image til GHCR.

```yaml
name: CI Build

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  build:
    uses: FINTLabs/fint-flyt-github-workflows/.github/workflows/java-app-build-image.yaml@main
    permissions:
      contents: read
      packages: write
    with:
      push-image: ${{ github.event_name == 'push' }}
    secrets: inherit
```

CD-workflows bruker `java-app-cd-deploy.yaml` med en repo-spesifikk JSON-liste
over `org`/`cluster`-targets. Den felles workflowen sjekker at CI-bygg på
`main` er vellykket, finner image fra bygget og deployer til target-listen.

Manuelle deploys kan bruke `java-app-deploy-aks.yaml` direkte med én `org`, én
`cluster` og image-referansen fra build-jobben.

Eksempel på `MD.yaml`:

```yaml
name: MD

on:
  workflow_dispatch:
    inputs:
      cluster:
        description: Velg cluster
        required: true
        type: choice
        options:
          - aks-beta-fint-2021-11-23
          - aks-api-fint-2022-02-08
      org:
        description: Velg organisasjon
        required: true
        type: choice
        options:
          - afk-no
          - fintlabs-no

jobs:
  build:
    uses: FINTLabs/fint-flyt-github-workflows/.github/workflows/java-app-build-image.yaml@main
    with:
      push-image: true
    secrets: inherit

  deploy:
    needs: build
    uses: FINTLabs/fint-flyt-github-workflows/.github/workflows/java-app-deploy-aks.yaml@main
    with:
      org: ${{ inputs.org }}
      cluster: ${{ inputs.cluster }}
      image-ref: ${{ needs.build.outputs.image-ref }}
    secrets: inherit
```

## Java library

Bibliotekrepoer kan bruke `java-library-ci.yaml` for vanlig Gradle-bygg.
Standard Java-versjon er 25, og standard kommando er `./gradlew build`.

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  build:
    uses: FINTLabs/fint-flyt-github-workflows/.github/workflows/java-library-ci.yaml@main
```

Publisering til Reposilite håndteres av
`java-library-publish-reposilite.yaml` på release-events. Workflowen setter
`RELEASE_VERSION` fra release-taggen og bruker `REPOSILITE_USERNAME` og
`REPOSILITE_PASSWORD` fra secrets.

## Dependency submission

`java-dependency-submission.yaml` løser hele det transitive avhengighetstreet
med Gradle og sender det til GitHub sitt Dependency Submission API. Uten dette
er avhengighetsgrafen tom for Gradle-pakker, og da har Dependabot ingenting å
matche sikkerhetsvarsler mot.

Denne workflowen trengs bare i **private** repoer. Public repoer får det samme
gratis gjennom GitHub sin innebygde «Automatic Dependency Submission», som ikke
er tilgjengelig for private repoer på Team-planen.

Merk at Dependabot sine versjonsoppdateringer virker uavhengig av dette —
de leser `build.gradle.kts` direkte. Det er kun sikkerhetsvarslene som er
avhengige av grafen.

Kall workflowen på push til `main`, slik at grafen alltid speiler default
branch. Et ukentlig schedule sørger for at grafen sendes inn på nytt selv i
perioder uten commits.

```yaml
name: Dependency Submission

on:
  push:
    branches:
      - main
  schedule:
    - cron: "24 3 * * 1"

jobs:
  submit:
    uses: FINTLabs/fint-flyt-github-workflows/.github/workflows/java-dependency-submission.yaml@main
    permissions:
      contents: write
```

Workflowen krever `contents: write` for å kunne sende inn grafen.
