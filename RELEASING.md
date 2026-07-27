# Releasing

Releases are cut by [GoReleaser](https://goreleaser.com). Pushing a tag matching `v*.*.*` runs the
[`release`](.github/workflows/release.yaml) workflow, which builds `linux/amd64` and `linux/arm64` binaries, publishes a
GitHub release with the archives and a checksum file, and pushes multi-arch container images to GHCR.

Version numbers and `CHANGELOG.md` are managed by
[release-please](.github/workflows/release-please.yaml), which opens a release pull request on every push to `main`.
Merging that pull request creates the tag, which in turn triggers the `release` workflow.

## Releasing from a fork

A fork can publish its own builds without any change to the configuration files.

Container images are published to the GHCR namespace of the repository being released, taken from the
`GITHUB_REPOSITORY` variable that GitHub Actions sets. A release from `kholisrag/fireactions` therefore publishes to
`ghcr.io/kholisrag/fireactions`, not to the upstream namespace. The built-in `GITHUB_TOKEN` is enough — the `release`
workflow already requests `packages: write`.

release-please is skipped on forks. It needs secrets that only the upstream repository has, and a fork running it would
produce version bumps that conflict with upstream on every sync. Tag fork releases by hand instead.

### Choosing a tag

Give fork releases a semver pre-release suffix so they can never collide with a tag upstream publishes later:

```bash
git tag -a v2.0.5-personal.1 -m "Fork build with repository scoped runner support"
git push origin v2.0.5-personal.1
```

GoReleaser marks any tag with a pre-release suffix as a GitHub pre-release, so fork builds don't present themselves as
the latest stable release. Container images are still tagged normally, and `:latest` in the fork's own namespace moves
to the most recent build.

### Installing a fork build

Point `install.sh` at the fork with `--fireactions-release-repository`:

```bash
./install.sh                                              \
  --fireactions-release-repository kholisrag/fireactions  \
  --fireactions-version 2.0.5-personal.1                  \
  --github-app-id <APP_ID>                                \
  --github-app-key-file <PATH_TO_PRIVATE_KEY.pem>         \
  --github-repository <OWNER>/<REPOSITORY>                \
  --containerd-snapshotter-device <DEVICE>
```

## Verifying the configuration locally

`goreleaser check` validates `.goreleaser.yml`, and a snapshot build produces the full set of artifacts under `dist/`
without publishing anything:

```bash
goreleaser check
GITHUB_REPOSITORY=<OWNER>/<REPOSITORY> goreleaser release --snapshot --clean --skip=publish
```

Building the container images requires a running Docker daemon with buildx. Add `--skip=docker` to build only the
binaries and archives.
