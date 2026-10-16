# Lotiva · Phase 1

An embedded Shopify app for assigning existing inventory to batches, following quantity changes, and identifying stock close to expiry.

**Status:** source implementation with automated validation. It has not been installed in a real Shopify development store in this environment. Use the acceptance checklist before treating it as production ready.

## What is included

- React Router 7 SSR app, TypeScript, Shopify App Bridge, and current Polaris web components.
- Shopify SDK authentication and Prisma session storage. Authenticated shop context determines every tenant query.
- PostgreSQL schema, versioned migration, composite tenant relations, quantity constraints, and a database trigger protecting the audit ledger.
- Cursor-paginated GraphQL inventory import: products, variants, tracked inventory items, location stock, on-hand and available quantities, store timezone.
- Batch creation with manufacture/expiry dates, supplier, location, and opening stock assignment.
- Signed quantity adjustments with mandatory reasons, serializable transactions, bounded retries, and idempotency keys.
- Overview, searchable batch register, batch detail, paginated inventory comparison and audit ledger, alert settings, empty states, loading feedback, and mobile layout.
- Authenticated webhooks, durable pending-sync markers, per-shop sync leases, retryable worker endpoint, uninstall handling, and compliance topic handlers.
- Unit and database integration tests, an ephemeral local test harness, and PostgreSQL CI configuration.

## Quantity semantics

Lotiva assigns **existing on-hand Shopify stock** to lots. Creating a batch does not receive new inventory into Shopify. Positive adjustments assign additional unassigned units; negative adjustments record dispatch, damage, or corrections in Lotiva. They do not change Shopify. A receipt must first be entered in Shopify and synced before its units can be assigned here.

`Difference = Shopify on hand − total batch quantity` at the same variant and location. Positive values are unassigned units; negative values require reconciliation. Available stock is displayed separately because it may exclude committed or reserved units. An external Shopify change can create a discrepancy after assignment. Lotiva does not automatically allocate sales to batches.

Expiry is stored as a calendar date and compared in the store's IANA timezone. A lot expiring today is marked “Expiring soon” until the next local day. Zero-balance lots are “Depleted” and excluded from active expiry alerts. Alerts are in-app; email, SMS and push delivery are outside Phase 1.

## Local setup

Requirements: Node 22.12+ (Node 24 recommended), npm, Docker or PostgreSQL 17, a Shopify developer account and development store, and Shopify CLI.

```sh
npm ci
cp .env.example .env
docker compose up -d db
npm run db:generate
npm run db:migrate
```

Set `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET`, and `SHOPIFY_APP_URL` in your local `.env`. Use the client ID and secret of a development app created in Shopify's Dev Dashboard. Keep secrets out of source control and frontend bundles. Update `client_id` in `shopify.app.toml` or link the existing app with Shopify CLI. See [development-store setup](docs/DEVELOPMENT_STORE.md).

```sh
npm install -g @shopify/cli
shopify app config link
npm run shopify:dev
```

Select the **development store**, approve the three read scopes, and open the app from Shopify Admin. Shopify CLI provides the tunnel URL; app URL and OAuth callback must match it. The web process must receive the environment values; do not assume a hosting provider loads `.env` automatically.

## Verification

```sh
npm run typecheck
npm run build
npm test
npm run test:local
```

`npm test` runs unit tests and skips database tests unless `RUN_DB_TESTS=1`. `test:local` creates a disposable PostgreSQL-compatible PGlite database, applies the actual SQL migration, and runs the whole suite. It uses TCP port 5433 in the same process environment and does not touch your normal database. Its wire adapter reconnects after expected database errors. It is not evidence of PostgreSQL's concurrent transaction behavior; PostgreSQL CI and development-store verification cover the remaining environment.

To run the database suite against **a disposable PostgreSQL database**:

```sh
DATABASE_URL=postgresql://user:password@localhost:5432/lotiva_test npm run db:migrate
DATABASE_URL=postgresql://user:password@localhost:5432/lotiva_test npm run test:integration
```

The tests insert isolated random tenant domains and clean them up. Never target a customer production database. [Validation details](docs/VALIDATION.md) distinguish completed checks from checks still needing Shopify or PostgreSQL infrastructure.

## Worker and deployment

1. Provision PostgreSQL with encryption at rest, TLS, backups, and a private network or restricted database access. Configure `DATABASE_URL` with provider-required TLS parameters.
2. Configure the environment values from `.env.example`. Generate a random `CRON_SECRET` of at least 32 characters.
3. Run `npm ci`, `npm run db:generate`, `npm run db:migrate`, and `npm run build`. Start with `npm start` behind HTTPS. `PORT` configures the web port.
4. Point `application_url`, auth redirect URL, and client ID in `shopify.app.toml` at the deployment. Run `npm run shopify:deploy` to release Shopify configuration. This command does not host the Node app.
5. Configure your hosting scheduler to POST `/internal/worker` every minute with `Authorization: Bearer <CRON_SECRET>`. This endpoint processes up to five pending shops sequentially and prunes webhook receipts older than 30 days. Choose a runtime supporting long-running requests for large catalogs; request timeouts leave the durable pending marker for retries. Large-scale queue infrastructure is outside this phase.
6. Monitor failed sync counts, pending age, database health, worker duration and HTTP failures. Complete [the acceptance checklist](docs/ACCEPTANCE.md).

Manual sync works independently of the worker. Webhooks queue a full reconciliation rather than applying unordered inventory deltas. A complete sync deactivates missing rows; a failed sync preserves missing rows and shows a visible warning. Existing rows may be refreshed during a partial import. The inventory comparison is a last-imported view, not a transactionally simultaneous Shopify snapshot. If webhooks arrive during a sync, their newer pending marker survives the completion.

## File map

| Path                               | Purpose                                       |
| ---------------------------------- | --------------------------------------------- |
| `app/shopify.server.ts`            | Shopify authentication and session storage    |
| `app/domain/batches.ts`            | Validation, stock arithmetic, timezone expiry |
| `app/services/batches.server.ts`   | Atomic batch mutations and idempotency        |
| `app/services/sync.server.ts`      | Paginated GraphQL import and sync leases      |
| `app/services/webhooks.server.ts`  | Durable webhook handling and redaction        |
| `app/services/workspace.server.ts` | Tenant-filtered, paginated views              |
| `app/routes/`                      | Authenticated loaders/actions and screens     |
| `app/components/`                  | Shared workspace UI                           |
| `prisma/schema.prisma`             | Data model including Shopify sessions         |
| `prisma/migrations/`               | Database constraints and ledger protection    |
| `tests/`                           | Domain, GraphQL and integration tests         |
| `.github/workflows/ci.yml`         | PostgreSQL checks on pushes and PRs           |

## Scope boundary

Phase 2 is not implemented: automatic order-to-lot allocation, FEFO picking, transfers, recalls, barcode scanning, purchase orders, billing, external notifications, analytics forecasting, and Shopify inventory writes.

See [security notes](docs/SECURITY.md) for tenant isolation, ledger retention, and deployment responsibilities.
