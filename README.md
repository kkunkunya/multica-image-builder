# multica-image-builder

Builds the self-host images for ai.ferryway.cc from the tip of
`jeff-kunkun/multica@kun` while that repository's GitHub Actions are disabled
(since 2026-10-08). Every 10 minutes it checks the tip and, if
`ghcr.io/kkunkunya/multica-{backend,web}:sha-<tip>` is missing, builds and
pushes both. The box pulls them with
`MULTICA_AUTOUPDATE_REGISTRY=ghcr.io/kkunkunya` in `/etc/multica/autoupdate.env`.

Once Actions run again on jeff-kunkun/multica, drop that line on the box and
delete this repository.
