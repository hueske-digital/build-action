# build-action

Builds a repository's image for linux/amd64 and linux/arm64 in parallel,
each on a GitHub runner of its own architecture (no emulation), and pushes
it to ghcr.io as one image for both. It is a reusable workflow, not an
action: only a workflow runs jobs in parallel.

```yaml
# .github/workflows/image.yml of a site's repository
name: image

on:
  push:
    branches: [main]

jobs:
  build:
    uses: hueske-digital/build-action/.github/workflows/build.yml@<commit> # v1.0.0
    permissions:
      contents: read
      packages: write

  # Optional: the host of the stack pulls the new image at once
  # (hueske-digital/deploy-action).
  deploy:
    needs: build
    runs-on: ubuntu-26.04
    timeout-minutes: 5
    permissions: {}
    steps:
      - uses: hueske-digital/deploy-action@2e880357b6dcbb764f0c1a7c936fd086be5b4410 # v1.0.0
        with:
          host: ${{ vars.DEPLOY_HOST }}
          stack: kunde
          secret: ${{ secrets.DEPLOY_SECRET }}
```

It runs three jobs: `setup` turns `platforms` into one build job each,
`build` builds and pushes each platform by its digest only, and `merge`
joins the digests into one image under its tags. The caller grants
`contents: read` and, to push, `packages: write`; the workflow asks for
nothing itself, so that a build without push runs with `contents: read`
alone.

| Input | Default | |
|---|---|---|
| `image` | `ghcr.io/<owner>/<repository>` in lower case | the image without a tag, on ghcr.io |
| `platforms` | `linux/amd64,linux/arm64` | `linux/arm64` alone builds only for arm64 |
| `context` | the whole repository | a directory of it |
| `file` | `Dockerfile` | relative to the context |
| `build-args` | | one `NAME=value` a line |
| `tags` | `latest`, and the tag's name for a pushed tag | one a line; `sha-<commit>` is always one |
| `push` | `true` | `false` builds without pushing, as for a pull request |

Outputs: `image` and `digest`, the digest of the image with both platforms
(its index), empty without push; a stack's entry may pin it
(`image@sha256:…`).

The build needs no checkout: it builds the commit of the run from its Git
context, which covers a private repository with the run's token. Each
platform keeps its layer cache in the repository's GitHub Actions cache
(`type=gha`, one scope per architecture). An input reaches a shell step only
through its environment, and `image` and every tag are checked before use.

The tests (`.github/workflows/test.yml`) build `test/Dockerfile` for both
platforms and for arm64 alone; on main they push it as this repository's
image and check that it holds both platforms, each built on its own
architecture, under `latest`, `sha-<commit>` and the digest the workflow
names.
