# pipeline-templates

Current release: **`1.0.4`** (uses build-ops-pipe `1.0.2`, image `provdev/platform:build-ops-pipe-11`; Docker 3072 MB, or 7128 MB in the `-2x` build pipelines)

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
    platform: pipeline-templates:1.0.4:deploy.yml

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

### Heavy builds (2x)

Apps whose image build needs more than 3072 MB of Docker memory (e.g. CRA frontends) import the
`-2x` variants of the two **build** pipelines. They run in a `size: 2x` step (8 GB, double build
minutes) with 7128 MB for Docker. Promotion and the PR check don't build, so they stay standard:

```yaml
pipelines:
  pull-requests:
    test:
      import: pr-check@platform
  branches:
    test:
      import: deploy-test-2x@platform
    prod:
      import: deploy-prod@platform
  custom:
    hotfix-build-production:
      import: hotfix-build-prod-2x@platform
```

Everyone stays on the latest release; pick memory by pipeline name, never by pinning an older tag.

`app-name` is the only thing an app sets. Every step reads it from the app's
`bitbucket-pipelines.yml` (an import cannot pass variables) and fails if it is missing.

Set these in the app repo (Repository settings -> Pipelines). Variables in this repo are not
passed to importing repos.

| Where | Variable |
|---|---|
| Repository (or workspace) | `DOCKERHUB_USERNAME`, `DOCKERHUB_READ_TOKEN` (secured, read-only; used by `pr-check` to look up app images), `GITOPS_BOT_EMAIL` |
| Workspace | `GITOPS_ACCESS_TOKEN` (secured, Repositories: Write on gitops-central-repo). Visible to every pipeline in the workspace |
| Deployments `Staging`, `Production` | `DOCKER_PASSWORD` (secured, push access) |

## Releasing

1. Change `deploy.yml` (e.g. bump `pipe-version` after a build-ops-pipe release) and update
   "Current release" at the top of this README. Merge to `main`.
2. Tag it with the next version: `git tag 1.0.3 && git push origin 1.0.3`.
3. Move each app's import (`pipeline-templates:<tag>:deploy.yml`) to the new tag.

The pipe image is public (`provdev/platform`), so steps need no `docker login` to pull it.

## Keeping app pipelines locked down

Developers can always edit their own `bitbucket-pipelines.yml`, so the import alone does not
stop changes. These do:

- **This repo:** write access for platform engineering only. Tags are the releases.
- **App repos, branch permissions on `test` and `prod`:** no direct pushes; merge only through a
  PR with a required platform approval, so an edited pipeline file never runs with the secrets.
- **Deployments:** keep `DOCKER_PASSWORD` on the Staging/Production environments only, and
  restrict those environments to the `test`/`prod` branches (and to admins where possible).
- **gitops-central-repo:** write access for the bot token only. Argo CD deploys only from there.
