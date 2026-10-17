# Plan 02: One host — PostgreSQL + local photos

Status: **implemented and checked locally**
Supersedes: `docs/plans/01-migration-cloudflare-to-pg-s3.md`
Authoritative brief: `docs/HOSTING_GUIDE.md`

## Goal

Take EV-GARAGE off Cloudflare, Vercel Blob, and any object-storage cloud. Run Next.js, PostgreSQL, and photo files on a single machine. Verify the app locally before it is published.

## Decision that replaces Plan 01

Yandex Object Storage, `minio`, and `@aws-sdk/*` are out. Photo bytes go to `PHOTOS_DIR` on the same host. PostgreSQL on the same host stores catalog data and photo metadata only. Scaling to separate storage is a later decision, not this release.

Catalog seed is `src/data/cars.ts` via `npm run db:seed`. There are no image files in the repo, and Cloudflare credentials are not part of this release, so KV photos are not imported. Galleries start empty.

Page HTML is rendered on each request (`force-dynamic`) so `next build` does not need a live database. The 15s in-memory catalog cache stays; errors and an empty result are not cached.

## In scope

- PostgreSQL schema, migration runner, seed from `src/data/cars.ts` and `src/data/suggestions.ts`
- `pg` pool; catalog, contacts, archive, admin videos, suggestion statuses
- Photo metadata in PostgreSQL; bytes under `PHOTOS_DIR`; existing upload/delete/reorder routes kept
- Remove OpenNext, Wrangler, `@vercel/blob`
- `docker-compose.yml` (Postgres, and a `full` profile with the app), Dockerfile, systemd unit, `.env.example`
- Local verification: lint, typecheck, production build, HTTP checks

## Out of scope

- Buying or SSH-ing into a production server, DNS cutover, copying live Cloudflare KV/D1/Blob (no credentials in this session)
- Deleting unused admin components that nothing calls

## Steps

1. Schema `migrations/postgres/0001_schema.sql` + `scripts/migrate-pg.mjs` + `scripts/seed-catalog.mjs`
2. `src/lib/pg.ts` and rewrite stores in `src/lib/` (`catalogDb`, `catalog`, `contactStore`, `archive`, `videosStore`, `suggestionStatus`, `photos`)
3. API routes return a logged 500 on database/disk failure
4. Remove Cloudflare/Blob from `package.json`, configs, and the deploy workflow; add CI for lint, typecheck, and build
5. Local Postgres, migrate, seed, build, and smoke the public pages, admin writes, and photo cycle

## Acceptance criteria

- [ ] `npm run db:migrate` applies once and is a no-op the second time; a changed checksum fails
- [ ] Seeded `/api/catalog` and `/api/brands` return brands from PostgreSQL
- [ ] Database down: `/api/catalog` is 500 and the body is not a successful empty catalog
- [ ] Admin contact, archive, video, and suggestion status writes survive a restart
- [ ] Photo upload of JPG/PNG/WEBP ≤ 4 MB creates a file under `PHOTOS_DIR` and a metadata row; oversize and other types do not
- [ ] Delete removes the file and the row; reorder changes gallery order
- [ ] No `getCloudflareContext`, `PHOTOS_KV`, `@vercel/blob`, `BLOB_READ_WRITE_TOKEN`, `minio`, or `@aws-sdk` in `src/` or `package.json`
- [ ] `npm run lint`, `npm run typecheck`, and `npm run build` pass

## Publishing and updates from GitLab

Status: **ready in the repo, not run** (needs a VPS and a GitLab project)

Push to the default branch updates the single host. GitLab CI tests the commit, then SSHs to the server, checks out that exact commit in `/opt/ev-garage`, and runs `scripts/deploy-host.sh`. That script rebuilds the compose stack (`postgres`, `app`, `caddy`). Postgres data and `PHOTOS_DIR` stay in Docker volumes. `.env` stays only on the server.

Files:

- `.gitlab-ci.yml` — `test` on every branch and merge request; `deploy` only on the default branch, and only after `SSH_HOST`, `SSH_USER`, `SSH_PRIVATE_KEY`, `SSH_KNOWN_HOSTS` are set
- `scripts/deploy-host.sh` — `docker compose --profile full up -d --build`, then wait for `/api/brands`
- `scripts/bootstrap-host.sh` — one-time root setup on Ubuntu: Docker, user `deploy`, read-only GitLab deploy key, CI SSH key
- `deploy/Caddyfile` — HTTPS in front of the app when `SITE_ADDRESS` is a domain

Deploy does not re-seed a non-empty catalog and does not delete photo files. Example passwords in `.env` abort the deploy.

### Acceptance criteria

- [ ] A push to the default branch on GitLab runs `test`, then `deploy`
- [ ] The server checkout equals `CI_COMMIT_SHA`
- [ ] `/api/brands` answers after deploy; existing photos and admin edits are still there
- [ ] A failed test does not SSH to the server
- [ ] Secrets are GitLab CI variables and `/opt/ev-garage/.env`, not files in git

The human steps are in `docs/HOSTING_GUIDE.md`, section «Что сделать для публикации».
