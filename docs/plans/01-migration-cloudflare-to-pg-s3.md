# Plan 01: Migration from Cloudflare to PostgreSQL + Yandex Object Storage

> **Superseded.** Публикация идёт на одном хосте: PostgreSQL и файлы фото на той же машине, без Yandex Object Storage и без `minio`. Актуальные требования: `docs/HOSTING_GUIDE.md`. Актуальный план: `docs/plans/02-single-host.md`.

Status: **superseded by plan 02**
Authoritative brief: `docs/HOSTING_GUIDE.md` (Этап 1–3; Этап 4–5 = VPS, out of scope for this plan)
Method: we execute **one step at a time**. After each step the Orchestrator stops and waits for user review. Next step starts only after user approval.

## Goal

Move EV-GARAGE off Cloudflare (D1 + KV + Vercel Blob + OpenNext) onto plain Node.js: PostgreSQL for all application data, Yandex Object Storage (S3 via `minio`) for photos, standard `next build` / `next start`.

## In scope

Этап 1 (all three blocks), Этап 2 (data-migration script), Этап 3 (local verification) of `docs/HOSTING_GUIDE.md`.

## Out of scope

Этап 4–5 (VPS provisioning, production deploy, domain switch) — separate plan after Этап 3 passes.

## Prerequisites

- Git branch for the work (guide line 11): create `migration` branch before Step 1.
- Credentials available when their step needs them (never committed):
  - local/test `DATABASE_URL` (Docker, Step 1)
  - test Yandex bucket + static key (Step 4)
  - `CLOUDFLARE_API_TOKEN` (D1:Edit + KV:Edit), `BLOUD… BLOB_READ_WRITE_TOKEN` (Step 6 export)
- `.env*` is gitignored; `.env.example` gets an explicit exception (guide line 67).

## Decisions (fixed, not to be re-litigated)

| Topic | Decision | Why |
|---|---|---|
| Migration layout | `migrations/postgres/*.sql` + tiny Node runner (`scripts/migrate-pg.mjs`, `schema_migrations` table, checksums) | coexists with legacy SQLite `migrations/0001_schema.sql`; zero extra deps (guide line 19) |
| Blob JSON docs → PG | normalized tables: `contact`, `archived_models`, `videos`, `suggestion_statuses` | keeps constraints/UNIQUE keys; avoids blob-style last-write-wins (guide line 18) |
| Static catalog fallback | remove (`catalog.ts`); DB error → log + throw; empty DB → `[]` + explicit warning | guide line 23 (no fake empty catalog / hidden fallback) |
| 15s catalog cache | keep, but only successful non-empty loads are cached | preserves behavior without masking errors |
| Photo bytes | Yandex Object Storage only; PG stores `object_key, photo_key(slug), position, content_type, ext, byte_size, updated_at` | guide line 30 |
| Object key scheme | `photos/{photoKey}/{index}.{ext}` where `photoKey = brandSlug-modelSlug` | matches existing `photoKey()` (photos.ts:338) and old KV keys → trivially mappable |
| S3 client | `minio` only; `@aws-sdk/*` forbidden | guide line 27 |
| Upload flow | admin gets server-signed `presignedPutObject` URL → browser PUT to Yandex → confirm endpoint writes PG metadata; type/size validated before issuing URL; on metadata failure delete the object | guide lines 31, 33 |
| Public reads | URLs built from `YANDEX_STORAGE_PUBLIC_BASE_URL`; `GET /api/photos/[slug]` kept as 302 redirect to the public URL (old links keep working) | guide lines 32, 46 |
| Build command | drop `--webpack` → plain `next build` (Next 16 = Turbopack default); revert to `--webpack` only if build fails, with reason noted | guide line 41 (standard Node.js build) |
| `next start` | keep `output: "standalone"` in `next.config.ts` (harmless with `next start`) | minimal diff |
| `@vercel/blob` | moved to `devDependencies` until Step 6 (export script reads production blobs), removed in Step 5 after stores are on PG | Step 6 still needs read access |
| Dead code (`/api/photos/upload`, `PhotoUploadRow.tsx`, `src/data/contact.ts`) | NOT deleted — question for user at Step 5 | guide line 46 (behavior preserved) |

---

## Steps

### Step 1 — Foundation: deps, env, local PostgreSQL, schema migrations

**Actions**
1. Create git branch `migration`.
2. `npm install pg` + `npm install -D @types/pg minio` (`npm.cmd` — PowerShell blocks `npm.ps1`).
3. Create `.env.example` — all vars from guide lines 88–99 + migration-source vars (`CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_D1_DATABASE_ID`, `CLOUDFLARE_KV_NAMESPACE_ID`, `BLOB_READ_WRITE_TOKEN`, `MIGRATION_CONFIRM_BUCKET`), placeholder values only.
4. Fix `.gitignore`: `.env*` + `!.env.example`, add `migration-data/`.
5. Create `docker-compose.yml` (postgres:17-alpine, db `ev_garage_test`, user/pass `ev`, port 5432, healthcheck) for local test DB.
6. Create `migrations/postgres/0001_schema.sql`:
   - translated catalog tables: `brands`, `models`, `model_colors`, `trims`, `suggestions` (IDENTITY PKs, `TIMESTAMPTZ DEFAULT now()`, `archived BOOLEAN`, `DOUBLE PRECISION` for REAL, all CHECK/FK/UNIQUE/indexes kept);
   - blob-doc tables: `contact`, `archived_models`, `videos`, `suggestion_statuses` (see Decisions);
   - photo metadata table: `photos(photo_key, position, object_key UNIQUE, content_type, ext, byte_size, updated_at, UNIQUE(photo_key, position))`.
7. Create `scripts/migrate-pg.mjs` (env loader, `DATABASE_URL` required, `schema_migrations` + checksums, `BEGIN/COMMIT` per file, explicit failures) + `"db:migrate"` script; add `"typecheck": "tsc --noEmit"`.

**Acceptance criteria**
- [ ] `npm run db:migrate` on fresh local DB applies all files; second run applies 0 (idempotent).
- [ ] Tampering with an applied file → runner fails with checksum error.
- [ ] `information_schema` shows catalog + doc + `photos` tables.
- [ ] `.env.example` committed; no real secrets anywhere; `git check-ignore .env.example` = not ignored.

**Verify:** `npm run db:migrate`, `npm.cmd run lint`, `npx tsc --noEmit`.

⛔ **Checkpoint — stop for user review.**

---

### Step 2 — PostgreSQL module + catalog on PG

**Actions**
1. `src/lib/pg.ts`: singleton `pg` Pool from `DATABASE_URL`, `query()` (log `code/message/detail`, rethrow), `withTransaction()`; missing `DATABASE_URL` → throw.
2. Rewrite `src/lib/catalogDb.ts`: remove `getCloudflareContext`; `loadBrandsFromPg(): Promise<Brand[]>` with the same 4 `SELECT … ORDER BY` queries (sort orders preserved); `archived` → boolean; no `null` returns.
3. Update `src/lib/catalog.ts`: call `loadBrandsFromPg`, **remove static `cars.ts` fallback**, keep 15s cache (errors/empty not cached); `cars.ts` stays as types/seed source only.

**Acceptance criteria**
- [ ] `git grep getCloudflareContext src/lib/catalogDb.ts` → empty.
- [ ] `git grep staticBrands src/lib/catalog.ts` → empty.
- [ ] DB down → `/api/catalog` returns 500 JSON + `[pg]` log; no empty/fake catalog.
- [ ] With seeded data: `/api/catalog`, `/api/brands`, brand/model/trim pages render from PG in original sort order.

**Verify:** `npm run lint`, `npx tsc --noEmit`, smoke with local PG.

⛔ **Checkpoint — stop for user review.**

---

### Step 3 — Blob JSON stores → PostgreSQL

**Actions**
1. Rewrite `src/lib/contactStore.ts` (`contact` upsert; no row → `DEFAULT_CONTACT`; DB error → throw), `src/lib/archive.ts` (`archived_models` insert/delete; ISO strings preserved; order by `archived_at`), `src/lib/videosStore.ts` (`videos` table; static `videoReviews` still merged from code; add = delete+insert in txn for append-order parity), `src/lib/suggestionStatus.ts` (`suggestion_statuses` upsert).
2. Remove all `@vercel/blob` imports and `BLOB_READ_WRITE_TOKEN` guards from these four files.
3. Add explicit 500 handling to API routes: `/api/contact`, `/api/archive`, `/api/videos`, `/api/suggestions`, `/api/catalog`, `/api/brands` (log + `{error}` JSON, never fake success).

**Acceptance criteria**
- [ ] `git grep "@vercel/blob\|BLOB_READ_WRITE_TOKEN" src/lib` → empty.
- [ ] Every store keeps its exported signatures (callers unchanged).
- [ ] Admin edits (contact, archive toggle, video add/remove, suggestion status) persist in PG and survive restart.
- [ ] DB error → 500 + log, not `{ok:true}`.

**Verify:** `npm run lint`, `npx tsc --noEmit`, manual admin smoke.

⛔ **Checkpoint — stop for user review.**

---

### Step 4 — Yandex Object Storage: storage module + photos rewrite

*Requires test bucket + static key (env).*

**Actions**
1. `src/lib/storage.ts` (server-only, `minio`): config from env (`YANDEX_STORAGE_ENDPOINT/REGION/BUCKET/ACCESS_KEY_ID/SECRET_ACCESS_KEY/PUBLIC_BASE_URL`), fail fast; methods `objectKeyForPhoto`, `publicUrlForObject`, `presignPutUrl`, `putObjectBuffer` (used by migration script only), `deleteObject`, `statObject`.
2. Rewrite `src/lib/photos.ts`: PG metadata + storage; keep exported API (`getPhotoMap`, `getPhotoGallery`, `listUsedIndices`, `reorderPhotos`, `putPhoto`, `deletePhoto`, `getPhoto`, `photoKey`, `PhotoMeta`); galleries ordered by `position`; URLs = `YANDEX_STORAGE_PUBLIC_BASE_URL/{object_key}?v={updated_at}`; all KV code and empty-catch fallbacks removed — errors surface (guide line 23).
3. New endpoints: `POST /api/photos/presign-upload` (admin auth, validate type in `image/jpeg|png|webp`, size ≤ 4 MB, slug/index limits, issue presigned PUT) and `POST /api/photos/confirm-upload` (admin auth, verify object via `statObject`, transactional PG upsert; on failure delete uploaded objects — guide line 33).
4. Rework routes: `GET /api/photos/[slug]` → 302 to public URL; `delete`, `reorder` unchanged externally; remove `upload`/`upload-fast` byte-proxying **only if user approves** (see Step 5 questions) — otherwise reimplement them as server-side presign+put+confirm wrappers so no route breaks.
5. Adapt `src/app/admin/PhotoGalleryEditor.tsx`: request presigned URLs → browser `fetch(PUT)` per file (parallel, with existing client compression) → confirm; keep limits, batch behavior, cache-buster via `v`.
6. Add photo-table migration file if not folded into `0001_schema.sql` (coordinate numbering).

**Acceptance criteria**
- [ ] `git grep "PHOTOS_KV\|getCloudflareContext" src` → empty.
- [ ] `git grep "@aws-sdk" package.json` → empty; `minio` present.
- [ ] Upload (jpg/png/webp ≤4 MB) via admin → object in test bucket + PG row; oversize/unsupported type → clear error, **no** metadata row, **no** orphan object.
- [ ] Delete removes object + row; reorder persists; gallery URLs resolve via public base URL.
- [ ] No permanent photo files written to local disk.

**Verify:** `npm run lint`, `npx tsc --noEmit`, full photo cycle on test bucket.

⛔ **Checkpoint — stop for user review.**

---

### Step 5 — Remove Cloudflare/OpenNext

*Only after Steps 2–4 removed every binding usage.*

**Actions**
1. `package.json`: remove `preview`/`deploy`/`cf-typegen` scripts; `build` → `next build`; `npm uninstall @opennextjs/cloudflare wrangler @vercel/blob`.
2. Delete `open-next.config.ts`, `wrangler.jsonc`, `.github/workflows/deploy-cloudflare.yml`; add `.github/workflows/ci.yml` (Node 22, `npm ci`, lint, `tsc --noEmit`, `next build` — no secrets, no deploy).
3. `.gitignore`: drop `.open-next`/`.wrangler`/`.dev.vars` lines (after confirming `.wrangler` local state is not needed — ask user).
4. Clean stale comments/refs: `src/middleware.ts` Vercel comment, `public/vercel.svg`, README rewrite (npm scripts, Node 22, VPS deploy pointer to `docs/HOSTING_GUIDE.md`).
5. Docs: delete/mark obsolete `docs/STAGE_A_D1.md`, `docs/skills/S7-deploy.md`, `docs/prompts/P6-deploy.md`; update `docs/README.md`, `docs/skills/README.md`, `S1/S4/S6/S9/S10`, `P0/P1/P5` CF references (ask user: delete vs mark obsolete).
6. Final grep gate over repo (excluding `docs/HOSTING_GUIDE.md`, `docs/plans/`, `docs/architecture/`, `docs/agents/`).

**Acceptance criteria**
- [ ] `git grep -i "cloudflare\|wrangler\|opennext\|PHOTOS_KV\|BLOB_READ_WRITE_TOKEN\|@vercel/blob" src package.json next.config.ts .github` → empty.
- [ ] `npm run lint`, `npx tsc --noEmit`, `npm run build` (plain, with `DATABASE_URL` set) all pass.
- [ ] Only scripts: `dev`, `build`, `start`, `lint`, `db:migrate`, `typecheck`.

⛔ **Checkpoint — stop for user review** (dead-code removal questions asked here).

---

### Step 6 — Data-migration script (Этап 2)

**Requires export credentials.** New `scripts/migrate-data.mjs` with subcommands `export | import [--dry-run] | verify`, snapshot dir `migration-data/` (gitignored):

1. **export** (read-only): D1 via REST (paged `SELECT *`, count assertions) → JSON; KV via REST (`list` prefix `cars/` + value bytes; `_index` docs parsed; legacy `cars/{slug}` handled) → files + key metadata; Vercel Blob 4 fixed JSON keys → files. Fallback: metadata rebuilt from `_index` if list omits it.
2. **import**: guard `MIGRATION_CONFIRM_BUCKET == YANDEX_STORAGE_BUCKET` + bucket name contains `test` (unless `--allow-non-test-bucket`); insert rows preserving slugs/relations/`sort_order`; upload photo files to `photos/{slug}/{i}.{ext}` with `statObject` size check; `position` = order inside `_index.items[]` (not numeric index!); blob docs → PG doc tables; PG writes in one transaction; idempotent (`ON CONFLICT`); `--dry-run` prints planned counts, writes nothing.
3. **verify**: re-count PG rows + bucket objects vs snapshot counts; mismatch → exit 5.
4. Report: counts per table/files/bytes, no secrets; exit codes 0–5; never writes to sources.

**Acceptance criteria**
- [ ] Dry-run and real run against local test DB + test bucket both exit 0 with matching counts.
- [ ] Re-run import → no duplicates.
- [ ] Source untouched (export issues only GET/SELECT).
- [ ] Forced failure (bad target creds) → non-zero exit, explicit error, no partial metadata rows.

⛔ **Checkpoint — stop for user review.**

---

### Step 7 — Local verification (Этап 3, items 3–12)

**Actions** (checklist, mostly manual + commands)
1. `npm ci`; `docker compose up -d`; `npm run db:migrate`.
2. `migrate-data export` → `import --dry-run` → `import` → `verify`; inspect report counts.
3. `npm run build` + `NODE_ENV=production PORT=3000 npm start`.
4. Check: home, brand/model/trim pages, `/compare`, `GET /api/catalog`, `GET /api/brands`, `GET /api/photos`.
5. Admin: login; edit contacts, archive toggle, videos, suggestion statuses → persist in PG.
6. Photo cycle via test bucket: upload → display → reorder → delete.
7. Confirm no local permanent photo files.
8. Error scenarios: DB stopped (explicit error), bad storage key (upload fails, no metadata row), 5 MB file (rejected), `image/gif` (rejected).
9. `npm run lint`; `npx tsc --noEmit`; grep gate from Step 5.

**Acceptance criteria:** all Этап 3 items pass; results written up as a short report for the user.

⛔ **Checkpoint — stop for user review.** After approval → next plan: Этап 4–5 (VPS).

---

## Risks & edge cases

- **Build now needs a reachable, seeded DB** (brand pages prerender) — accepted; matches guide ordering (migrate → build).
- **KV list metadata availability** — export falls back to rebuilding from `_index` (lossy for exotic contentType); verify on first export run.
- **Photo `position` vs KV index** — display order = `_index.items[]` array order; migration must not sort by numeric index.
- **CORS on test/prod bucket** for browser PUT — infrastructure step, documented in Step 4/Этап 4.
- **`archived_models` FK to `models`** — archiving unknown slug now errors (was silent); accepted, admin only archives catalog models.

## Open questions for the user (non-blocking until their step)

1. Step 5: delete dead code (`/api/photos/upload`, `PhotoUploadRow.tsx`, `src/data/contact.ts`) or keep?
2. Step 5: Cloudflare docs — delete or mark obsolete?
3. Step 6: do you already have `CLOUDFLARE_API_TOKEN` (D1:Edit + KV:Edit) and `BLOB_READ_WRITE_TOKEN`?
4. Step 2: is remote D1 still the catalog truth, or can it diverge from `src/data/cars.ts` seed? (Affects what Step 6 exports.)

## Handoff

- Architector: `docs/architecture/` docs to build before Step 1–2 — `storage-map.md`, `pg-schema.md`, `env-vars.md`; before Step 4 — `photo-flow.md`, `api-contracts.md`.
- Tester: riskiest criteria — no-fallback error paths (Steps 2–3), photo failure/rollback semantics (Step 4), grep gates (Step 5), count reconciliation (Step 6).
