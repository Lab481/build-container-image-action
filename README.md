# build-container-image-action

GitHub composite action for building container images the Lab481 way

## Inputs

See `action.yml` for up to date documentation.

```yaml
inputs:
  registry_name:
    description: registry name (default ghcr.io)
    required: false
    default: "ghcr.io"
  registry_username:
    description: registry username
    required: true
  registry_password:
    description: registry password
    required: true
  image_repository:
    description: container image repository
    required: true
  dockerfile_path:
    description: path to Dockerfile (default ./Dockerfile)
    default: "./Dockerfile"
    required: false
  build_args:
    description: build args
    default: "build args to be supplied to the build. Specify as list of newline delimited KEY=VALUE tuples"
    required: false
  build_context:
    description: path to build context (default .)
    default: "."
    required: false
  build_target:
    description: sets the target stage to build
    default: ""
    required: false
  push_to_registry:
    description: push to container registry (true/false)
    required: true
  buildx_version:
    description: Buildx version to install, independent of the runner's preinstalled one (default v0.38.1)
    default: "v0.38.1"
    required: false
  use_build_cache:
    description: Read and write the BuildKit layer cache in the GitHub Actions cache (true/false, default true)
    default: "true"
    required: false
  buildx_driver:
    description: Buildx driver; "docker" reuses the persistent runner's local layer cache (needs use_build_cache false)
    default: "docker-container"
    required: false
  skip_login:
    description: Skip Docker registry login
    default: "false"
    required: false
```

## Example Usage

```yaml
name: Build container image

on:
  push:
    branches:
      - main
    tags:
      - v**
  pull_request:
    branches:
      - main

jobs:
  build-container:
    name: Build container image
    runs-on: ubuntu-latest
    # required when using GitHub registry
    permissions:
      packages: write
      contents: read
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Build and push container image
        uses: Lab481/build-container-image-action@main # or use a valid tag
        with:
          registry_username: ${{ github.repository_owner }}
          registry_password: ${{ secrets.GITHUB_TOKEN }}
          image_repository: ${{ github.repository }}
          dockerfile_path: deployment/Dockerfile.prod
          push_to_registry: ${{ github.event_name != 'pull_request' }}
```

## static-site-check

Gate for nginx static-site images before they are pushed: a smoke test under
the Mainframe pod restrictions (read-only root, UID 101, no capabilities) and a
Trivy scan that fails on fixable vulnerabilities of `trivy_severity`. Build the
image with `load: true` first, run this, and push only afterwards. Repos using it
let Renovate automerge nginx base image patch and digest updates once it passes.

```yaml
- name: Check image
  uses: Lab481/build-container-image-action/static-site-check@main
  with:
    image: my-site:ci
    expect_text: "Page title"
    ok_paths: |
      /favicon.png
```

Trivy only sees CVEs of Debian packages; nginx itself comes from nginx.org's
package and is not matched. nginx security releases reach the sites through
Renovate's version updates, not through this scan. The Trivy DB is kept in the
runner's `~/.cache/trivy`, since the self-hosted runners are persistent.
