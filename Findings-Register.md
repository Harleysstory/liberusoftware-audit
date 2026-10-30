# Liberu Ecosystem — Findings Register (Quick Reference)

Inspection date: 2026-10-18 | Method: public web/static review | Full detail: [Liberu-Ecosystem-Audit.md](Liberu-Ecosystem-Audit.md)

## Security findings

| ID | Sev | Conf | Component | Summary | Effort |
|----|------|------|-----------|---------|--------|
| SEC-01 | MEDIUM | H | boilerplate .env.example | `APP_DEBUG=true`, `SESSION_ENCRYPT=false` defaults risk prod misconfig | 0.5 pd |
| SEC-02 | HIGH | H | crm-laravel composer.json | `twilio/sdk: "*"` unbounded constraint | 0.25 pd |
| SEC-03 | MEDIUM | H | all apps composer.json | `minimum-stability: dev` widens supply chain | 0.25 pd |
| SEC-04 | MEDIUM | H | composer-installer | Path-traversal protection VERIFIED good; remaining risk = no release signing + single-point trust | 2 pd |
| SEC-12 | **CRITICAL** | H | module-organizations-teams, module-roles-permissions, module-identity-core | Foundation modules owning tenancy/authz/authn ship with ZERO tests (`tests/`=`.gitkeep`); docs mandate 100% coverage + 2-team negative tests | 10–20 pd |
| SEC-13 | HIGH | M | module-organizations-teams | Documented `BelongsToTenant` trait NOT found; actual trait is `BelongsToTeam` in genealogy-core only | 3–5 pd |
| SEC-16 | MEDIUM | H | boilerplate/crm/accounting/genealogy | **CVE-2026-45070** (symfony/mime email header injection) confirmed in all 4 apps via `composer audit` (executed 2026-10-30) | 0.25 pd |
| SEC-05 | HIGH | H | crm vs genealogy | Inconsistent tenancy: stancl/tenancy vs Jetstream-Team global scopes | 5–15 pd |
| SEC-06 | MEDIUM | M | module-identity-core | Thin identity core; authorisation delegated to host → per-app drift | 3–5 pd |
| SEC-07 | LOW | H | boilerplate routes | No CSRF exemption; auth middleware correct (informational, no action) | — |
| SEC-08 | MEDIUM | L | module fleet | API scopes, webhook signatures, upload guards mandated by docs but unverified | 3–5 pd |
| SEC-09 | MEDIUM | M | CI/CD | No Dependabot/CodeQL observed; branch protection unverified | 1 pd |
| SEC-10 | MEDIUM | M | legacy liberu-* orgs | Abandoned repos public; website links to non-existent repos | 1–2 pd |
| SEC-11 | MEDIUM | M | all apps | Rate-limit config for auth/API endpoints unverified | 1–2 pd |

## Engineering / product findings

| ID | Sev | Component | Summary |
|----|------|-----------|---------|
| ENG-01 | CRITICAL | liberusoftware org | **~2,400 public repos** (verified 2026-10-29); the vast majority are `module-*` packages with 0 stars/forks/issues/PRs — extreme breadth, near-zero community validation of depth; sustainability/maintainability risk at planetary scale |
| ENG-06 | HIGH | crm/accounting product modules | Many product modules have NO declared composer `require` (no php/illuminate) → `composer install` may produce runtime failures |
| ENG-07 | MEDIUM | most modules/adapters | No README + no tests for most product modules & presentation adapters; violates documented "every module has a complete README" rule |
| ENG-08 | MEDIUM | ecosystem | Maturity is highly stratified (genealogy=real+tested; CRM/accounting=thin; foundation=real but UNTESTED); "uniform quality" claim not borne out |
| ENG-11 | LOW | H | boilerplate-laravel | 6 test failures on Windows (theme path-separator); tests assume forward-slash paths |
| POS-03 | INFO | H | ecosystem | No hardcoded production credentials found; test files use fake keys to verify secret handling |
| ENG-02 | HIGH | CI tests.yml | Host coverage gate `--min=99` applies to `app/` only; module gates not seen running in host CI |
| ENG-03 | MEDIUM | org descriptions | 4 apps still labelled "Laravel 12" vs documented Laravel 13 baseline |
| ENG-04 | MEDIUM | crm composer.json | Package name `liberu-crm/crm-laravel` ≠ documented `liberusoftware/` namespace |
| ENG-05 | MEDIUM | liberusoftware.com | "View on GitHub" buttons link to `liberu-{x}/{x}-laravel` repos that don't exist |
| SAAS-01 | MEDIUM | portfolio | SaaS billing exists only in genealogy (Cashier); no shared entitlement engine |
| SAAS-02 | MEDIUM | all apps | Observability/backup/DR runbooks not observed; packages available only |
| SAAS-03 | MEDIUM | docs | "Liberu platform" cross-product app is docs/planning scope only — no shipped repo |
| UIUX-01 | CANNOT ASSESS | apps | No running app tested; staging Lighthouse/axe audit required |

## Confirmed Critical findings (measured 2026-10-30)
- **SEC-12** — 5 of 8 security-critical foundation modules (tenancy, authz, authn, files, API access) have ZERO tests. Confirmed by local filesystem scan.
- **SEC-14** — `BelongsToTeam` trait is **fail-open**: silently skips tenant scoping if `team_id` column absent → cross-tenant data leak. Full source read.
- **SEC-15** — Tenancy isolation is genealogy-product-specific (not shared foundation); only 88 files use it; other apps have no isolation.
- **ENG-01/09** — 2,395 repos; 846 (35%) have NO source code; 1,911 (80%) have ZERO tests.
No confirmed CRITICAL *network-exploitable* vulnerability (no dynamic testing performed), but tenant isolation is unproven and fail-open by design.
