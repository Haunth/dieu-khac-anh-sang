# Komodo deployment runbook

This repository is the source of truth for the application image and Compose topology. Komodo should clone the repository, validate the Compose configuration, build the image on the target server, and deploy it.

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

## 2. Komodo stack configuration

Configure the `dieu-khac-anh-sang` stack as follows:

- Repository: `thainq3127/dieu-khac-anh-sang`
- Branch: `main`
- Compose file: `compose.yaml`
- Run build before deploy: enabled
- Auto pull images: enabled
- Webhook: enabled
- Force deploy on webhook: disabled
- Destroy before deploy: disabled
- Environment file: `.env`

Recommended pre-deploy command:

```sh
docker compose --env-file .env -f compose.yaml config --quiet
```

This makes a deployment fail before changing containers when a required variable is missing or the Compose file is invalid.

## 3. Required production variables

The canonical list is `.env.example`. The minimum application variables are:

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
- `NEXT_PUBLIC_ASSET_BASE_URL`
- `POSTGRES_PASSWORD`

`BASIC_AUTH_*`, GA4 variables, and `GEMINI_API_KEY` are optional.

`NEXT_PUBLIC_ASSET_BASE_URL` and `NEXT_PUBLIC_GA_ID` are build-time values. Changing either requires rebuilding the image, not only restarting the container.

## 4. Database preflight

The current project uses raw SQL and does not expose a migration command in `package.json`. Before the first application deploy:

1. Back up the PostgreSQL volume/database.
2. Confirm the expected tables already exist.
3. Review SQL files under `supabase/` and `migrations/` before applying them. Do not blindly execute every file in lexical order because the repository contains historical and migration-specific scripts.
4. Add an idempotent migration runner before enabling unattended deployments.

Until a migration runner exists, database schema changes remain a manual, reviewed step.

## 5. Deployment sequence

1. Merge a reviewed pull request into `main`.
2. Back up the database when schema changes are included.
3. Trigger the Komodo deployment.
4. Wait for both `db` and `app` healthchecks to pass.
5. Verify the public page, `/admin/login`, one database-backed page, and one media upload.
6. Record the deployed commit SHA.

## 6. Rollback

1. Pin the Komodo stack to the last known-good commit.
2. Rebuild and redeploy.
3. Restore the database only when the failed release included an irreversible schema/data migration.

Never use `destroy` as a routine rollback action because it removes containers and increases the chance of accidental data loss.

## 7. Next improvements

- Add an idempotent migration command and a one-shot `migrate` Compose service.
- Build immutable registry images tagged with the Git commit SHA.
- Add CI checks for `pnpm lint`, `pnpm build`, secret scanning, and Compose validation.
- Replace plaintext environment-based admin authentication with hashed users stored in the database.
