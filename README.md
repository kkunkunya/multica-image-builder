# multica-image-builder

Stand-in release pipeline for `jeff-kunkun/multica` while that repository's
GitHub Actions are disabled (since 2026-10-08). Two workflows, both on a
10-minute schedule plus `workflow_dispatch`:

| Workflow | Watches | Produces |
| --- | --- | --- |
| `build.yml` | tip of `kun` | `ghcr.io/kkunkunya/multica-{backend,web}:sha-<tip>` self-host images for ai.ferryway.cc. The box pulls them with `MULTICA_AUTOUPDATE_REGISTRY=ghcr.io/kkunkunya` in `/etc/multica/autoupdate.env`. |
| `cli-linux.yml` | newest `v*` tag | Linux CLI/daemon archives for that tag: `multica-cli-<ver>-linux-{amd64,arm64}.tar.gz`, the legacy `multica_linux_{amd64,arm64}.tar.gz`, and `checksums.txt`, built with the source tree's own `.goreleaser.yml` so names, `--version` and checksums match an official release. |
| `build-test.yml` | manual only (`-f ref=<branch-or-sha>`) | `ghcr.io/kkunkunya/multica-test-{backend,web}:sha-<sha>` for a throwaway test instance. Separate package names, so no self-host box follows them. |
| `build-cutover.yml` | manual only (`-f sha=<40-char commit>`) | `ghcr.io/kkunkunya/multica-{backend,web}:sha-<sha>` with no `:kun` tag, prepared before a signed cutover points the box at another branch. Inert until autoupdate targets that commit. |

Once Actions run again on jeff-kunkun/multica, drop that env line on the box
and delete this repository.

## Linux CLI packages

### Where they land

- **Default (no secret):** on this repository's Release named after the tag,
  e.g. `https://github.com/kkunkunya/multica-image-builder/releases/tag/v0.6.3-test.5`.
  The Release notes say which source commit it was built from.
- **With the `SOURCE_RELEASE_TOKEN` secret** (a token that may write Releases
  on `jeff-kunkun/multica`): on the source repository's own Release for the
  tag, next to the Desktop assets. Nothing else changes; `multica update`
  then needs no extra configuration.

The job is idempotent: it skips when the target Release already has all five
assets, and `--clobber`s on rerun. Backfill an older tag with:

```bash
gh workflow run cli-linux.yml -R kkunkunya/multica-image-builder -f tag=v0.6.3-test.5
```

### Upgrade a server

`multica update` reads `jeff-kunkun/multica` by default (`MULTICA_RELEASE_REPO`
overrides it) and only installs from a Release that carries `checksums.txt`,
which the archives here do. While the packages live on this repository:

```bash
# newest Release on this repository (test tags included)
MULTICA_RELEASE_REPO=kkunkunya/multica-image-builder multica update
multica --version        # tag version + source commit
```

Or pin a tag by hand (also works without an installed CLI):

```bash
TAG=v0.6.3-test.5; ARCH=amd64   # or arm64
curl -fsSL "https://github.com/kkunkunya/multica-image-builder/releases/download/$TAG/multica-cli-${TAG#v}-linux-$ARCH.tar.gz" -o /tmp/multica.tar.gz
curl -fsSL "https://github.com/kkunkunya/multica-image-builder/releases/download/$TAG/checksums.txt" | grep "multica-cli-${TAG#v}-linux-$ARCH.tar.gz" | sed 's#  .*#  /tmp/multica.tar.gz#' | sha256sum -c
tar -xzf /tmp/multica.tar.gz -C /tmp multica && sudo install -m 0755 /tmp/multica /usr/local/bin/multica
multica --version
```

Restart the daemon afterwards if one is running (`multica daemon status`).
When `SOURCE_RELEASE_TOKEN` is set, replace the repository in the URLs above
with `jeff-kunkun/multica` and drop the `MULTICA_RELEASE_REPO=` prefix.
