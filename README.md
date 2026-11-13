# pipeline-templates

Shared Bitbucket Pipelines for Providus apps: build, PR image check, promotion and the
GitOps update, all done by [build-ops-pipe](https://bitbucket.org/nndukwe/build-ops-pipe).
App repos import these pipelines instead of copying them. Bitbucket shared pipeline
configurations need the Premium plan.

## Using it in an app repo

The app's whole `bitbucket-pipelines.yml`:

```yaml
definitions:
  app-name: my-app   # image tag prefix and gitops folder: environments/<env>/<app-name>
  imports:
    platform: pipeline-templates:1.0.0:deploy.yml

pipelines:
  pull-requests:
    test:
      import: pr-check@platform
  branches:
    test:
      import: deploy-test@platform
    prod:
      import: deploy-prod@platform
  custom:
    hotfix-build-production:
      import: hotfix-build-prod@platform
```

`app-name` is the only thing an app sets. Every step reads it from the app's
`bitbucket-pipelines.yml` (an import cannot pass variables) and fails if it is missing.

Set these in the app repo (Repository settings -> Pipelines). Variables in this repo are not
passed to importing repos.

| Where | Variable |
|---|---|
| Repository | `DOCKERHUB_USERNAME`, `DOCKERHUB_READ_TOKEN` (secured, read-only), `GITOPS_ACCESS_TOKEN` (secured), `GITOPS_BOT_EMAIL` |
| Deployments `Staging`, `Production` | `DOCKER_PASSWORD` (secured, push access) |

## Releasing

1. Change `deploy.yml` (e.g. bump `pipe-version`) and merge to `main`.
2. Tag it: `git tag 1.1.0 && git push origin 1.1.0`.
3. Move each app's import to the new tag.

## Keeping app pipelines locked down

Developers can always edit their own `bitbucket-pipelines.yml`, so the import alone does not
stop changes. These do:

- **This repo:** write access for platform engineering only. Tags are the releases.
- **App repos, branch permissions on `test` and `prod`:** no direct pushes; merge only through a
  PR with a required platform approval, so an edited pipeline file never runs with the secrets.
- **Deployments:** keep `DOCKER_PASSWORD` on the Staging/Production environments only, and
  restrict those environments to the `test`/`prod` branches (and to admins where possible).
- **gitops-central-repo:** write access for the bot token only. Argo CD deploys only from there.
