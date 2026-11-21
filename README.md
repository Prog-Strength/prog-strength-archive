# prog-strength-archive

The cold archive of [Prog Strength](https://github.com/Prog-Strength), which
was deprecated and torn down on 2026-11-21. Everything needed to bring the
project back lives in one private, versioned S3 bucket managed by
[Jaws](https://github.com/Jaws-Platform/jaws). This repository holds only the
Jaws project file and this description. **No data is stored in git.**

| | |
| --- | --- |
| Bucket | `jaws-prog-strength-archive-backups` |
| Account | JWall Engineering production (`570327089884`), profile `jaws-prod` |
| Region | `us-east-1` |
| Retention | Versioned, nothing expires |
| Managed by | Jaws `projects.prog-strength-archive` in `config.yaml` and this repository's `jaws.yaml` |

## Layout

Everything is under `data/`, the dataset's prefix.

```text
data/
  MANIFEST.sha256                 SHA-256 of every file below, relative to data/
  db/
    snapshots/<date>/app.db       plain, consistent SQLite copies (online backup API)
    snapshots/<date>/telemetry.db
    litestream/                   the Litestream replica, exactly as it was in S3
      prog-strength-api/prod/       app.db
      prog-strength-telemetry/prod/ telemetry.db
  media/
    activity-photos/              was s3://prog-strength-activity-photos
    activity-videos/              was s3://prog-strength-activity-videos
    avatars/                      was s3://prog-strength-avatars
    tcx-uploads/                  was s3://prog-strength-tcx-uploads
  terraform/
    prog-strength-infra-prod.tfstate   the application stack's last state, before destroy
  secrets/
    key-names.json                the KEY NAMES (never values) of each backend secret
```

The latest `db/snapshots/<date>/` is the one to restore from. It was taken
with the API stopped, so it holds every write the service ever accepted.

The media keys are unchanged from the original buckets, so the paths stored in
`app.db` still resolve once the files are copied into buckets with the same
layout.

## Restoring

```sh
export AWS_PROFILE=jaws-prod
B=s3://jaws-prog-strength-archive-backups/data

# Everything, then verify it.
aws s3 sync "$B" ./prog-strength-archive
(cd prog-strength-archive && shasum -a 256 -c MANIFEST.sha256)

# The database alone, from the plain snapshot.
aws s3 cp "$B/db/snapshots/<date>/app.db" ./app.db

# Or from the Litestream replica, after copying db/litestream/ into a bucket
# litestream can read:
litestream restore -o app.db s3://<bucket>/prog-strength-api/prod
```

To run the stack again: recreate the application infrastructure from
`prog-strength-infra` (in an account other than the management account), copy
`media/*` into the new buckets, put `app.db` and `telemetry.db` in the host's
`compose/api/data/`, and re-seed the secrets named in `secrets/key-names.json`
with fresh credentials. Treat the original credentials as revoked.

## Uploading the archive

Done once, at teardown, by the owner with admin credentials (`jaws-prod`):
no CI role writes this bucket.

```sh
aws s3 sync ./final-backup/ s3://jaws-prog-strength-archive-backups/data/
```
