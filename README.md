# cresset-tools GitLab CI/CD components

GitLab counterparts of the cresset-tools GitHub Actions
([setup-bougie](https://github.com/cresset-tools/setup-bougie),
[sconce-publish](https://github.com/cresset-tools/sconce-publish),
[sconce-upload](https://github.com/cresset-tools/sconce-upload),
[magequery-action](https://github.com/cresset-tools/magequery-action)),
packaged as [CI/CD components](https://docs.gitlab.com/ee/ci/components/):
one template per tool under `templates/`, each with typed `spec:inputs`.

No Docker images and no Node: every job curl-installs a small prebuilt
binary via its cargo-dist installer, so cold starts are fast on any runner.

## Using the components

On **gitlab.com** (via the mirrored catalog project):

```yaml
include:
  - component: gitlab.com/cresset-tools/components/sconce-publish@1.0.0
    inputs:
      repo: acme/packages
      url: https://repo.example.com
```

On **any instance, including self-managed** (straight from GitHub — pin a tag
in the URL; `inputs` work the same because the templates carry `spec:inputs`):

```yaml
include:
  - remote: https://raw.githubusercontent.com/cresset-tools/gitlab-components/1.0.0/templates/sconce-publish.yml
    inputs:
      repo: acme/packages
      url: https://repo.example.com
```

`include:component` only resolves projects on the *same* GitLab instance, so
self-managed instances use the `remote:` form (or mirror this repo onto the
instance and reference it there).

## Components

### `bougie`

A hidden base job, `.bougie` — the `setup-bougie` counterpart. Extend it to
get bougie on PATH with PHP interpreters, Composer packages, and service
runtimes cached between runs. The cache lives inside the project dir via
`BOUGIE_HOME`/`BOUGIE_CACHE` (GitLab won't cache paths outside it) under one
stable, mutable key — deliberately *not* keyed on `composer.lock`: GitLab
has no prefix-fallback restore, so a lock-hash key would drop the whole
cache on every lock change, while bougie happily reuses a partial cache and
fetches only the delta.

```yaml
include:
  - component: gitlab.com/cresset-tools/components/bougie@1.0.0
    inputs:
      version: 0.44.1   # optional; defaults to latest

test:
  extends: .bougie
  image: debian:stable-slim  # anything with curl + tar
  script:
    - bougie sync
    - bougie run -- vendor/bin/phpunit
```

| Input | Default | Description |
| --- | --- | --- |
| `version` | `latest` | bougie version to install. |
| `php-version` | _(empty)_ | Optional PHP to pre-install for use outside a project. |
| `telemetry` | `false` | `true` / `local` opt in to anonymous telemetry. |
| `cache-key` | `bougie` | Cache key; segment per architecture/image if runners are mixed. |

The install runs from `before_script`; a job that defines its own
`before_script` replaces it — re-run it first with
`!reference [.bougie, before_script]`.

### `sconce-publish`

Adds a `sconce-publish` job (runs on tag pipelines by default) that pushes a
Composer package to a sconce repository with **zero stored secrets**: GitLab
mints an OIDC JWT into `$SCONCE_ID_TOKEN` via `id_tokens:`, and `sconce
publish` exchanges it for a short-lived publish token.

```yaml
include:
  - component: gitlab.com/cresset-tools/components/sconce-publish@1.0.0
    inputs:
      repo: acme/packages
      url: https://repo.example.com
```

| Input | Default | Description |
| --- | --- | --- |
| `repo` | _(required)_ | Target sconce repository, `<org>/<repo>`. |
| `url` | _(required)_ | Base URL of the sconce wire server. |
| `dir` | `.` | Package directory (contains `composer.json`). |
| `version` | `$CI_COMMIT_TAG` | Version to publish. |
| `audience` | `sconce` | OIDC audience the publish policy expects. |
| `sconce-version` | `latest` | sconce CLI version to install. |
| `stage` | `deploy` | Pipeline stage. |
| `image` | `buildpack-deps:curl` | Job image (needs curl + tar). |

### `sconce-upload`

Adds a `sconce-upload` job that pushes a database snapshot (e.g. a
`.jibsdump`) as an environment's new `latest`, authenticated the same
zero-secret way. It ships without `rules:`, so add your own trigger:

```yaml
include:
  - component: gitlab.com/cresset-tools/components/sconce-upload@1.0.0
    inputs:
      repo: acme/packages
      url: https://repo.example.com
      file: dumps/prod.jibsdump
      environment: production

sconce-upload:
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
```

Inputs match `sconce-publish`, minus `dir`/`version`, plus required `file`
and `environment`.

### `magequery`

Adds a `magequery` job that runs a static check or query against a Magento 2
/ Mage-OS project. By default it first installs bougie and runs `bougie sync`
so magequery sees a populated `vendor/` tree.

```yaml
include:
  - component: gitlab.com/cresset-tools/components/magequery@1.0.0
    inputs:
      args: modules --check
```

| Input | Default | Description |
| --- | --- | --- |
| `version` | `latest` | magequery version to install. |
| `args` | `doctor` | Any magequery command and its arguments. |
| `root` | `.` | Magento project root. |
| `sync` | `true` | Install bougie + `bougie sync` first. |
| `cache-key` | `bougie` | Cache key for the bougie tree (shared with the bougie component). |
| `stage` | `test` | Pipeline stage. |
| `image` | `buildpack-deps:curl` | Job image (needs curl + tar). |

## sconce OIDC setup (once per repo)

The publish/upload components need `sconce` ≥ the release that understands
`$SCONCE_ID_TOKEN` (GitLab's pre-issued JWT; older CLIs only speak GitHub
Actions' request-URL flow). On the server side, add a CI policy trusting the
GitLab project:

```sh
sconce ci-policy add --repo acme/packages --provider gitlab \
  --issuer https://gitlab.com --audience sconce \
  --claim project_path=acme/shop --capability publish
```

Useful GitLab claims to match on: `project_path`, `namespace_path`, `ref`,
`ref_type`, `ref_protected`. On self-managed GitLab, `--issuer` is the
instance URL — and the sconce server must be able to reach it to fetch the
JWKS. If it can't (instance behind a VPN), fall back to a stored token: set a
masked `SCONCE_PUBLISH_TOKEN` CI/CD variable and the CLI uses it instead of
OIDC.

## Publishing to the CI/CD Catalog (maintainers)

The canonical source lives on GitHub; gitlab.com consumption goes through a
push mirror:

1. Create `gitlab.com/cresset-tools/components`, push this repo to it (a
   GitHub Actions workflow that force-pushes `main` + tags does fine).
2. In the GitLab project: Settings → General → set a description, then
   **Visibility** → toggle *CI/CD Catalog resource*.
3. Tag semver releases as bare `1.0.0` (the catalog requires plain semver
   tags). The `create-release` job in `.gitlab-ci.yml` turns the tag into a
   catalog release on the GitLab side.

`include:remote` consumers just pin the same tags in the raw URL — no
catalog involved.
