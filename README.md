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
| Multi-arch | `linux/amd64` and `linux/arm64` via QEMU + Docker Buildx |
| Atomic multistage | All targets must build before *any* image is pushed |
| Cache | GHA cache shared between build and push phases |
| Defaults | Works with GHCR out of the box (`github.actor` / `GITHUB_TOKEN`) |

#### Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `image-name` | ✅ | — | Image name without registry prefix, e.g. `owner/myapp` |
| `registry` | | `ghcr.io` | Container registry |
| `dockerfile` | | `Dockerfile` | Path to the Dockerfile |
| `context` | | `.` | Docker build context path |
| `targets` | | `[""]` | JSON array of Dockerfile target names. `[""]` builds the default (final) stage. |
| `platforms` | | `linux/amd64,linux/arm64` | Comma-separated target platforms |
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

All targets are built first. Only if every build succeeds are the images pushed.

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