# .github

Organization-wide reusable GitHub Actions workflows.

---

## Workflows

### `docker-build-push.yml` – Build and publish Docker images

Builds and pushes multi-architecture Docker images (`linux/amd64` + `linux/arm64`)
to a container registry (GHCR by default).

**Key features**

| Feature | Detail |
|---|---|
| Multi-arch | `linux/amd64` and `linux/arm64` built on **native** GitHub-hosted runners — no QEMU |
| Atomic multistage | All targets must build before *any* image is pushed |
| Manifest merge | Per-platform images are pushed by digest and joined into a single multi-arch manifest |
| Defaults | Works with GHCR out of the box (`github.actor` / `GITHUB_TOKEN`) |

#### How it works

The workflow has two phases:

1. **`build` job** (matrix: `target × platform`, `fail-fast: true`):  
   Each job runs on the **native** runner for its platform (`ubuntu-latest` for amd64, `ubuntu-24.04-arm` for arm64).  
   The image is built and pushed to the registry **by digest only** (no tag). If any target fails to build, the entire matrix is cancelled and nothing is ever tagged.

2. **`push` job** (`needs: build`, matrix: `target`):  
   Runs only when **every** build job succeeds. Downloads all per-platform digests for the target and calls `docker buildx imagetools create` to assemble and push the final multi-arch manifest with the computed tags.

This gives the "all-or-nothing" atomicity guarantee across all targets.

#### Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `image-name` | ✅ | — | Image name without registry prefix, e.g. `owner/myapp` |
| `registry` | | `ghcr.io` | Container registry |
| `dockerfile` | | `Dockerfile` | Path to the Dockerfile |
| `context` | | `.` | Docker build context path |
| `targets` | | `[""]` | JSON array of Dockerfile target names. `[""]` builds the default (final) stage. |
| `platforms` | | `["linux/amd64", "linux/arm64"]` | JSON array of target platforms |
| `tags` | | SHA + `latest` on default branch | Tag rules for `docker/metadata-action` |
| `registry-username` | | `github.actor` | Registry username |

#### Secrets

| Name | Required | Description |
|---|---|---|
| `registry-password` | | Registry password/token. Falls back to `GITHUB_TOKEN` (suitable for GHCR). |

#### Required permissions for the calling workflow

```yaml
permissions:
  contents: read
  packages: write   # needed to push to GHCR
```

#### Examples

**Single-stage image**

```yaml
jobs:
  publish:
    uses: federico-busetti-io/.github/.github/workflows/docker-build-push.yml@main
    with:
      image-name: ${{ github.repository }}
    permissions:
      contents: read
      packages: write
```

**Multistage image (atomic)**

All targets are built first on native runners. Only if every build succeeds are the
multi-arch manifests created and pushed.

```yaml
jobs:
  publish:
    uses: federico-busetti-io/.github/.github/workflows/docker-build-push.yml@main
    with:
      image-name: ${{ github.repository }}
      targets: '["base", "app"]'
      # Pushes ghcr.io/<org>/<repo>/base:<tag> and ghcr.io/<org>/<repo>/app:<tag>
    permissions:
      contents: read
      packages: write
```

**Custom registry**

```yaml
jobs:
  publish:
    uses: federico-busetti-io/.github/.github/workflows/docker-build-push.yml@main
    with:
      image-name: myorg/myapp
      registry: registry.example.com
      registry-username: myuser
    secrets:
      registry-password: ${{ secrets.MY_REGISTRY_TOKEN }}
    permissions:
      contents: read
      packages: write
```