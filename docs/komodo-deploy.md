# Komodo deployment runbook

This repository is the source of truth for the Dockerfile and Compose topology. Image building and application deployment are intentionally separate Komodo resources:

```text
GitHub main
  -> Komodo Build
  -> registry.vrtour.vn/admin/dieu-khac-anh-sang
  -> Komodo Stack pulls image
  -> healthcheck and smoke test
```

`compose.yaml` is image-only. It must never contain an inline `build:` section.

## 1. One-time security actions

1. Revoke and rotate the Google service-account private key that was previously committed in `.env.example`.
2. Keep production secrets only in Komodo environment/secret variables. Never commit them to Git.
3. Use separate random values for `POSTGRES_PASSWORD`, `AUTH_SECRET`, `ADMIN_PASSWORD`, and `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`.

Recommended generators:

```sh
openssl rand -base64 48  # AUTH_SECRET
openssl rand -base64 32  # NEXT_SERVER_ACTIONS_ENCRYPTION_KEY
openssl rand -base64 24  # passwords
```

## 2. Komodo Build resource

Create a dedicated Build resource, suggested name: `dieu-khac-anh-sang-build`.

Recommended configuration:

- Repository: `thainq3127/dieu-khac-anh-sang`
- Branch: `main`
- Build context: `.`
- Dockerfile: `Dockerfile`
- Registry: `registry.vrtour.vn`
- Image repository: `admin/dieu-khac-anh-sang`
- Push image after a successful build: enabled
- Webhook from GitHub: enabled

Build arguments:

- `NEXT_PUBLIC_ASSET_BASE_URL`
- `NEXT_PUBLIC_GA_ID`
- `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`

`NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` must be the same stable value used by the running Stack. Do not generate a new value on every build.

Recommended tagging policy:

- Push an immutable tag containing the Git commit SHA.
- Optionally also update `latest` for the normal deployment lane.
- Use the immutable SHA tag for audit and rollback.

The Build resource owns source checkout, Docker build logs, image tags, and registry push. The Stack must not clone source or build Docker images.

## 3. Komodo Stack resource

Configure the existing `dieu-khac-anh-sang` Stack as follows:

- Compose source: repository `compose.yaml`, or tracked file contents synchronized from it
- Run build before deploy: disabled
- Auto pull images: enabled
- Destroy before deploy: disabled
- Environment file: `.env`
- `APP_IMAGE`: full registry image reference

Normal lane example:

```env
APP_IMAGE=registry.vrtour.vn/admin/dieu-khac-anh-sang:latest
```

Pinned release or rollback example:

```env
APP_IMAGE=registry.vrtour.vn/admin/dieu-khac-anh-sang:<git-commit-sha>
```

Recommended pre-deploy validation:

```sh
docker compose --env-file .env -f compose.yaml config --quiet
```

The Stack lifecycle is limited to pulling and running images. A deploy failure must not trigger a source build inside the Stack.

## 4. Required Stack runtime variables

The canonical list is `.env.example`. Minimum runtime variables:

- `APP_IMAGE`
- `DATABASE_URL`
- `AUTH_URL`
- `AUTH_SECRET`
- `ADMIN_EMAIL`
- `ADMIN_PASSWORD`
- `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`
- `S3_ENDPOINT`
- `S3_BUCKET`
- `S3_ACCESS_KEY_ID`
- `S3_SECRET_ACCESS_KEY`
- `POSTGRES_PASSWORD`

`BASIC_AUTH_*`, GA4 server variables, and `GEMINI_API_KEY` are optional.

`NEXT_PUBLIC_ASSET_BASE_URL` and `NEXT_PUBLIC_GA_ID` are Build resource arguments, not Stack variables. Changing either requires a new image build and push.

## 5. Database preflight

The current project uses raw SQL and does not expose a migration command in `package.json`. Before the first application deploy:

1. Back up the PostgreSQL volume/database.
2. Confirm the expected tables already exist.
3. Review SQL files under `supabase/` and `migrations/` before applying them. Do not blindly execute every file in lexical order because the repository contains historical and migration-specific scripts.
4. Add an idempotent migration runner before enabling unattended schema deployment.

Until a migration runner exists, database schema changes remain a manual, reviewed step.

## 6. Release sequence

1. Merge a reviewed pull request into `main`.
2. Trigger `dieu-khac-anh-sang-build`.
3. Confirm the image was pushed successfully and record its immutable SHA tag.
4. Set `APP_IMAGE` to the desired tag, or use the updated `latest` lane.
5. Deploy the `dieu-khac-anh-sang` Stack with image pulling enabled.
6. Wait for both `db` and `app` healthchecks to pass.
7. Verify the public page, `/admin/login`, one database-backed page, and one media upload.
8. Record the Git commit SHA and image digest.

A Komodo Procedure may later orchestrate Build then Stack deploy, but the resources must remain separate.

## 7. Rollback

1. Set `APP_IMAGE` to the last known-good immutable SHA tag.
2. Redeploy the Stack without rebuilding.
3. Restore the database only when the failed release included an irreversible schema/data migration.

Never use `destroy` as a routine rollback action because it removes containers and increases the chance of accidental data loss.

## 8. Next improvements

- Add an idempotent migration command and a reviewed one-shot migration job.
- Add CI checks for `pnpm lint`, `pnpm build`, secret scanning, and Compose validation.
- Add a Komodo Procedure: Build -> verify pushed image -> Stack deploy -> smoke test.
- Replace plaintext environment-based admin authentication with hashed users stored in the database.
