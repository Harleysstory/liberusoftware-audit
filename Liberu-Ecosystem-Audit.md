# Liberu Software Ecosystem — Comprehensive Technical & Commercial Audit

**Prepared by:** Independent Consultant (Principal Software Architect / Laravel Security Auditor / SaaS Strategist)
**Inspection dates:** 2026-10-18 (web-based), 2026-10-29 (org listing + module deep-dive), 2026-10-30 (local filesystem fleet scan of 2,395 cloned repos)
**Primary sources:** Public GitHub repositories under `github.com/liberusoftware`, `github.com/liberu-*` orgs, `liberusoftware.com`, `liberusoftware/documentation`, and **local clone at `C:\Users\harle\liberu-software`** (2,395 repos, read access)
**Review method:** Phase 1–2: static web inspection. Phase 3 (2026-10-30): **direct filesystem measurement** of all 2,395 cloned repositories using shell tools and file reads. No tools (composer/phpstan/pest) were executed; no live systems were tested.
**Status:** Phase One (Discovery) substantially complete. Phases Two–Eight based on risk-based sampling of the highest-impact components. Coverage honestly reported below.

---

## 0. Evidence and Access Limitations (read first)

This audit is constrained by access. The following were **not** performed and the report makes no claim that they were:

- Repositories were **not cloned**. No `composer install`, `composer audit`, `phpstan`, `pint`, `pest`, Semgrep, or CodeQL was executed by this auditor. Tool outputs cited are from CI workflow *definitions* observed in-repo, not from this auditor's own runs.
- No running application was deployed or tested. **No dynamic testing, OWASP ZAP scan, or Lighthouse run was performed.**
- Source inspection was limited to files readable via the GitHub web interface / raw.githubusercontent.com. The `liberusoftware` organisation contains **~2,400 public repositories** (verified 2026-10-29); deep line-by-line review of the full module fleet was not feasible; risk-based sampling was used.
- Commit hashes and exact line references are cited where the GitHub interface exposed them; some line-level detail is necessarily approximate.
- Dependency vulnerability status reflects `composer audit` as configured in CI, not an independent audit of `composer.lock`.

Findings are marked: **[VERIFIED]** (directly observed in source/web), **[SUSPECTED]** (inferred from evidence, not confirmed), **[RECOMMENDATION]**, or **[CANNOT ASSESS]**.

---

## 1. Executive Summary

Liberu Software is a UK-based organisation building a large portfolio of open-source Laravel applications (accounting, CRM, billing, ecommerce, genealogy, CMS, real estate, maintenance, social network, browser game, control panel) plus a shared modular Composer-package foundation. The documentation repository (`liberusoftware/documentation`) describes an **ambitious, sophisticated, well-reasoned architecture**: domain-neutral core packages, provider adapters, presentation adapters (API/Filament/Livewire/React/Vue/Nuxt/React Native/Flutter), strict dependency-direction rules, manifest-driven modules, a custom Composer installer, and progressive adoption profiles from personal to enterprise.

The **documentation quality is genuinely high** — among the better open-source architecture specs I have reviewed. The architectural *intent* is sound and modern.

However, there is a **substantial gap between documented architecture and verifiable implementation reality**, and several material risks affect commercial SaaS viability:

| # | Headline finding | Severity |
|---|---|---|
| 1 | **Two parallel, inconsistent ecosystems** — the active `liberusoftware/` org (Laravel 13, modular) vs the legacy `liberu-*` orgs (Laravel 10/12, monolithic). The public website links to the *legacy* orgs, which are partially abandoned. | HIGH |
| 2 | **Inconsistent tenancy model across applications** — genealogy uses Jetstream-Team-based tenancy (per docs); CRM pulls in `stancl/tenancy`. The documented "single canonical tenancy" is not uniformly implemented. | HIGH |
| 3 | **Extreme module breadth without verified depth** — the `liberusoftware` organisation now contains **~2,400 public repositories** (verified via the GitHub org repositories listing on 2026-10-29). Most are `module-*` packages, multiplied by per-module presentation-adapter repos (`-api`, `-filament`, `-livewire`, etc.). The vast majority show 0 stars/forks/issues/PRs — minimal community engagement. The documentation mandates 100% owned-PHP coverage per package, but host CI only enforces 99% on the ~5 composition classes in `app/`. Module-level coverage gates were not observed running in the inspected host CI. | HIGH |
| 4 | **Supply-chain and dependency hygiene gaps** — `twilio/sdk: "*"` (unbounded) in CRM; a custom Composer installer plugin is trusted via `allow-plugins`; `minimum-stability: dev` is set in application composer files. | MEDIUM |
| 5 | **Thin identity/security core** — `module-identity-core` owns only "authentication policy, identifier normalisation, security events" and explicitly delegates authorisation to the host. Real authn/authz weight rests on Jetstream defaults + per-host composition, which varies per app. | MEDIUM |
| 6 | **SaaS billing exists in only one app** (genealogy, via Cashier/Stripe). Other apps have no demonstrated subscription/billing/offboarding implementation. | MEDIUM |
| 7 | **No observed centralised observability/backup/DR evidence in code** beyond package *availability* (Horizon, Pulse, spatie/backup are dependencies). Operational readiness is unverified. | MEDIUM |

**Overall recommendation: CONDITIONAL GO** (leaning toward NO-GO for SaaS until SEC-12/14/15 are resolved) — see §13 for conditions.

**Measured fleet statistics (2026-10-30, local scan):** 2,395 repos; 35% have no source code; 80% have zero tests; the tenancy-isolation trait is genealogy-specific and fail-open; only CRM uses `stancl/tenancy`. See [Fleet-Wide-Measured-Statistics.md](Fleet-Wide-Measured-Statistics.md).

---

## 2. Repository Inventory and Coverage Matrix

### 2.1 Active organisation: `liberusoftware` (the modular, Laravel 13 ecosystem)

| Repository | Classification | Laravel | Commits (approx.) | Stars | Inspection |
|---|---|---|---|---|---|
| `liberusoftware/documentation` | Documentation hub | n/a | 95 | — | README, INSTALL, MODULES, TENANCY, SECURITY, API, ADOPTION inspected |
| `liberusoftware/boilerplate-laravel` | Application — foundation/SaaS boilerplate | 13 / PHP 8.5 | 703 | 202 | README, composer.json, .env.example, routes, CI workflow inspected |
| `liberusoftware/accounting-erp-laravel` | Application — accounting/ERP | 13 / PHP 8.5 | 1,294 | — | README, composer.json inspected |
| `liberusoftware/crm-laravel` | Application — CRM | 13 / PHP 8.5 | 2,064 | — | README, composer.json, tests/, modules/ directory listing inspected |
| `liberusoftware/genealogy-laravel` | Application — genealogy (reference tenancy impl) | 13 / PHP 8.5 | — | — | README, composer.json inspected |
| `liberusoftware/billing-laravel` | Application — hosting billing (cPanel/Plesk/DA) | 12 / PHP 8.5 | — | — | Description only |
| `liberusoftware/browser-game-laravel` | Application — PBBG | 12 / PHP 8.5 | — | — | Description only |
| `liberusoftware/control-panel-laravel` | Application — control panel | 12 / PHP 8.5 | — | — | Description only |
| `liberusoftware/cms-laravel` | Application — CMS | 13 / PHP 8.5 | — | — | Description only |
| `liberusoftware/ecommerce-laravel` | Application — ecommerce | 13 / PHP 8.5 | — | — | Description only |
| `liberusoftware/social-network-laravel` | Application — social network | 13 / PHP 8.5 | — | — | Description only |
| `liberusoftware/real-estate-laravel` | Application — real estate | 13 / PHP 8.5 | — | — | Description only |
| `liberusoftware/maintenance-laravel` | Application — CMMS | 13 / PHP 8.5 | — | — | Description only |
| `liberusoftware/enterprise-sap-laravel` | Application — SAP-style suite | 13 / PHP 8.5 | — | — | Description only; docs note "documentation/planning scope" |
| `liberusoftware/module-identity-core` | Shared module — identity/auth policy | 13 | 27 | — | README, composer.json, module.json inspected |
| `liberusoftware/module-audit`, `module-application`, `module-roles-permissions`, `module-organizations-teams`, `module-jetstream-bridge`, `module-files-media`, `module-webhooks`, `module-notifications`, `module-settings`, `module-localization-core`, `module-search`, `module-scheduler-queues`, `module-observability`, `module-feature-flags`, `module-import-export`, `module-integrations`, `module-api-access`, `module-sessions-devices`, `module-two-factor-authentication`, `module-profiles`, `module-module-manager`, `composer-installer`, `package-testbench`, `*-contracts`, themes, etc. | Shared modules / themes / tooling | 13 | — | — | Referenced in boilerplate/genealogy composer.json; not individually inspected |
| `liberusoftware/accounting-*`, `crm-*`, `genealogy-*` product modules | Product modules | 13 | — | — | Referenced in host composer.json; CRM modules/ listing observed (300+ dirs) |

### 2.2 Legacy organisations: `liberu-accounting`, `liberu-billing`, `liberu-crm`, etc.

| Org | Repos observed | Status |
|---|---|---|
| `liberu-accounting` | `.github`, `accounting-nuxt` (Vue3/Nuxt3 frontend), `accounting-old` (Laravel 10 backend) | Legacy/abandoned backend; no `accounting-laravel` repo exists despite website linking to it |
| `liberu-crm` | (profile only, mentions Laravel 12) | Legacy |
| `liberu-billing` | `.github` only | Essentially empty |
| `liberu-genealogy`, `liberu-ecommerce`, etc. | Linked from website | Not all verified; pattern suggests legacy orgs |

**[VERIFIED]** The `liberusoftware.com` website's "View on GitHub" buttons link to `liberu-{domain}/{domain}-laravel` repositories (e.g., `liberu-accounting/accounting-laravel`), but those repositories **do not exist** in the `liberu-accounting` org (which contains `accounting-old` and `accounting-nuxt` instead). The active code lives under `liberusoftware/`. This is a broken/misleading public funnel.

### 2.3 Coverage matrix

| Phase | Repos inspected method | Coverage | Outstanding |
|---|---|---|---|
| 1. Discovery | Web listing of org + docs | ~14 application repos + documentation + 1 module deep-inspected; **~2,400 module/product repos** across the org (verified count, not individually inspected) | Full per-repo inspection infeasible; legacy orgs not exhaustively checked |
| 2. Architecture | composer.json + docs | Boilerplate, CRM, accounting, genealogy, identity-core | Other apps; per-module source |
| 3. Security | .env.example, routes, identity-core, tenancy docs | Defaults, tenancy model, identity boundary | Per-module policies, migrations, controllers, webhooks, file uploads |
| 4. UI/UX | None (no running app) | Cannot assess | Requires staging deployment |
| 5. SaaS | README billing section (genealogy), composer deps | Billing in 1 app; tenancy inconsistent | Billing/offboarding in other apps; monitoring/DR |
| 6. Tooling | CI workflow file | CI definition observed; not executed | Actual tool runs |
| 7/8. Findings/Roadmap | Synthesis | Sampled high-risk | Full per-module confirmation |

---

## 3. Architecture and Dependency Map

### 3.1 Documented architecture (from `documentation/`)

```
Application (composition root) ──requires──> Module packages (Composer, type: liberu-module)
        │                                          │
        │ composes panels, routes, bindings,        │ domain rules, migrations, policies, events, jobs
        │ entitlement, infrastructure               │
        ▼                                          ▼
   Filament/Livewire/API/React/Vue/Nuxt/RN/Flutter adapters (presentation, never domain)
        │
        ▼
   Provider adapters (Stripe, Twilio, Mailchimp, Google, SAML, etc.) implement contracts
```

- **Foundation packages:** identity, organisations/teams, roles/permissions, audit, settings, localisation, currency, notifications, files, queues, observability, feature-flags, import/export, integrations, webhooks, sessions/devices, 2FA, profiles, module-manager.
- **Custom Composer installer** (`liberusoftware/composer-installer`) installs `liberu-module` packages into `/modules/{name}` and `liberu-theme` into `/themes/{name}`. `/modules` is **tracked in Git** (not gitignored) so deployments show exact installed code.
- **Dependency direction:** presentation → domain → contracts → foundation. Domain must not depend on Filament/themes/provider SDKs. Apps must not be depended upon by packages.
- **Tenancy (documented):** Jetstream `Team` as the tenant; `BelongsToTenant` trait adds `team` relation + global scope + create-time `team_id` stamping; Filament 5 app-panel tenancy; team-scoped export paths; queue-worker context reset; two-team negative tests required.

### 3.2 Verified implementation observations

- **[VERIFIED]** Boilerplate `composer.json` requires ~50+ `liberusoftware/*` packages plus Laravel 13, Filament 5, Livewire 4, Jetstream 5, Sanctum 4, Octane 2, Horizon 5, Pulse, Reverb, spatie/permission 7, spatie/passkeys, spatie/backup, filament-shield. This matches the documented foundation.
- **[VERIFIED]** CRM `composer.json` package name is `liberu-crm/crm-laravel` (vendor prefix `liberu-crm`), **not** `liberusoftware/crm-laravel`. Inconsistent vendor namespacing vs. the documented "everything under `liberusoftware/`".
- **[VERIFIED]** CRM requires `stancl/tenancy: ^3.10` — a different tenancy approach than the documented Jetstream-Team model. Genealogy does **not** require `stancl/tenancy` and relies on Jetstream teams + Cashier. **Tenancy implementation diverges between applications.**
- **[VERIFIED]** `routes/api.php` in boilerplate is empty (`<?php` only) — all API surface is contributed by module packages. Consistent with documented design.
- **[VERIFIED]** `routes/web.php` dashboard route uses `auth:sanctum` + Jetstream auth session; super-admin redirects to admin panel with tenant resolution via `getDefaultTenant($panel)`.
- **[VERIFIED]** `.env.example` defaults: `APP_DEBUG=true`, `SESSION_ENCRYPT=false`, `DB_CONNECTION=sqlite`, `CACHE_STORE=database`, `QUEUE_CONNECTION=database`. These are Laravel-stock dev defaults; acceptable for a boilerplate but **must be hardened for SaaS**.
- **[SUSPECTED]** Many module repos are very recently created (e.g., `module-identity-core` created 2026-08-01, 27 commits). This suggests a recent large-scale modularisation effort; maturity/depth of the fleet is unproven.

### 3.3 Trust boundaries (as designed)

1. **Unauthenticated** → public routes / login.
2. **Authenticated user** → Jetstream session; team membership gates team resources.
3. **Team tenant** → global scope + policies isolate team-owned records.
4. **API consumers** → Sanctum tokens with scopes (per API.md; scope enforcement is per-module, not verified).
5. **Admin panel** → Filament `admin` panel, `isSuperAdmin()` gate (observed in web.php).
6. **Background workers** → queue jobs must carry tenant context (documented; not verified in code).
7. **Webhook receivers** → signature verification mandated by docs; implementation not verified.

---

## 4. Technical Maturity Assessment

| Dimension | Assessment | Evidence | Verdict |
|---|---|---|---|
| **Documentation** | Excellent intent, prescriptive, ADR-driven | MODULES.md, API.md, TENANCY.md, ADOPTION.md | Strong |
| **Modular decomposition** | Ambitious; ~200+ packages | composer.json repos, CRM modules listing | Breadth high; depth unverified |
| **Laravel conventions** | Modern stack (L13, PHP8.5, Filament5, Livewire4) | composer.json | Strong on paper |
| **Version consistency** | Mixed | 4 apps still "Laravel 12" in descriptions (billing, browser-game, control-panel) vs Laravel 13 baseline | Gap |
| **Testing (definition)** | Pest 5, architecture tests, coverage policy | MODULES.md §26, CI workflow | Strong policy |
| **Testing (enforcement)** | Host CI gates `--min=99` on `app/` only | tests.yml comment | Weak host gate; module gates not seen running here |
| **Static analysis** | PHPStan level 9 configured | CI workflow, phpstan.neon present | Good (if actually green) |
| **CI/CD** | composer validate/audit, pint, phpstan, tests, codecov; `permissions: contents: read` | tests.yml | Good |
| **Dependency hygiene** | `twilio/sdk: "*"`, `minimum-stability: dev` | CRM composer.json | Gap |
| **Tenancy uniformity** | Divergent (stancl vs Jetstream teams) | CRM vs genealogy composer.json | Gap |
| **SaaS billing** | Only genealogy (Cashier) | README | Gap for other apps |
| **Observability/DR** | Packages available (Horizon/Pulse/Telescope/backup); no runbook/manifest observed | composer deps | Unverified |

---

## 5. Security Findings Register

Severity: CRITICAL / HIGH / MEDIUM / LOW / INFORMATIONAL. Confidence: H/M/L.

| ID | Sev | Repository / Component | Finding | Evidence | Confidence | Remediation | Effort |
|---|---|---|---|---|---|---|---|
| SEC-01 | MEDIUM | `boilerplate-laravel` `.env.example` | `APP_DEBUG=true` and `SESSION_ENCRYPT=false` defaults. If a deployer copies `.env.example` to production unchanged, debug pages leak stack traces and sessions are not encrypted at rest. | `.env.example` lines (verified) | H | Ship a `.env.production.example` with `APP_DEBUG=false`, `SESSION_ENCRYPT=true`, `CACHE_STORE=redis`, `QUEUE_CONNECTION=redis`; add a deployment pre-flight check that fails on insecure prod defaults. | 0.5 day |
| SEC-02 | HIGH | `crm-laravel` | `twilio/sdk: "*"` unbounded version constraint. Any major Twilio SDK release can break installs or introduce behavioural changes; also blocks `composer audit` reproducibility. | CRM composer.json `require` (verified) | H | Pin to a caret range (`^7.0` or current) and update via CI. | 0.25 day |
| SEC-03 | MEDIUM | All apps | `minimum-stability: dev` with `prefer-stable: true` in application composer.json. Allows dev-stability packages to resolve; combined with many first-party VCS repos this widens the supply chain. | boilerplate + genealogy composer.json (verified) | H | Set `minimum-stability: stable` for applications; reserve `dev` for development only. | 0.25 day |
| SEC-04 | MEDIUM | `composer-installer` plugin | Custom Composer installer is trusted via `allow-plugins` and writes into `/modules`. A compromise of the installer repo could place arbitrary code in every consumer. No evidence of signature verification beyond Composer's VCS trust. | composer.json `allow-plugins`, MODULES.md §6.1 | M | Pin installer to exact commit/tag in lockfile; require signed releases (GPG/Sigstore); add a CI check that the installed `/modules` tree matches lockfile-resolved sources. | 2 days |
| SEC-05 | HIGH | `crm-laravel` vs `genealogy-laravel` | Inconsistent tenancy: CRM uses `stancl/tenancy` (database/schema-per-tenant model) while genealogy uses Jetstream `Team` global scopes (per TENANCY.md). A module built assuming one model may leak data under the other. The documented "single canonical tenancy" is not enforced. | CRM composer.json `stancl/tenancy`, genealogy composer.json (no stancl), TENANCY.md "Reference implementation: genealogy" | H | Pick ONE tenancy model; migrate CRM off `stancl/tenancy` (or formally adopt it everywhere); add architecture tests forbidding the unused approach; add cross-team negative tests in every tenant-aware module. | 5–15 days |
| SEC-06 | MEDIUM | `module-identity-core` | Identity module owns only "authentication policy, identifier normalisation, security events" and explicitly states "Authorization remains the host application's responsibility." Authn/authz weight is delegated to Jetstream defaults + per-host composition, which varies per application. Inconsistent policy enforcement risk across apps. | module-identity-core README (verified) | M | Define a shared authorizer contract + default policies in the identity/roles-permissions modules; provide reference policy implementations consumed by all apps; add tests proving every tenant-owned model has a policy. | 3–5 days |
| SEC-07 | LOW | `boilerplate-laravel` routes | `routes/web.php` dashboard uses `auth:sanctum` + Jetstream auth session — correct. No CSRF exemption observed. Welcome route is unauthenticated and harmless. | routes/web.php (verified) | H | No action; informational confirmation. | — |
| SEC-08 | MEDIUM | Module fleet | Per-module API scope enforcement, webhook signature verification, file-upload path-traversal guards, and mass-assignment protection are *mandated by documentation* but were **not verified** in source for any module. The documentation is strong; the implementation is unproven at scale. | API.md §, SECURITY.md (docs only) | L (cannot confirm absence or presence) | Audit a representative sample (files-media, webhooks, api-access, import-export) directly; add automated checks (e.g., Larastan mass-assignment rules, test that every controller has a policy). | 3–5 days sampling |
| SEC-09 | MEDIUM | CI/CD | CI uses `actions/checkout@v5`, `setup-php@v2`, `codecov-action@v5`. `permissions: contents: read` is set — good. No `dependabot.yml` or `codeql-action` observed in the inspected workflow. Dependency-update automation not confirmed. | tests.yml (verified) | M | Add Dependabot/Renovate config and, if on GitHub Advanced Security, CodeQL. Confirm branch protection + required status checks via org settings (not web-visible). | 1 day |
| SEC-10 | MEDIUM | Legacy `liberu-*` orgs | Abandoned/old repositories (e.g., `accounting-old` Laravel 10) remain public. If any contain hardcoded credentials or vulnerable dependencies they reflect on the brand and may be indexed. | `liberu-accounting/accounting-old` listing (verified) | M | Audit legacy repos for secrets/credentials, archive or transfer, and remove public website links to them. | 1–2 days |
| SEC-11 | MEDIUM | All apps | No evidence of rate-limiting configuration for API/auth endpoints beyond Sanctum defaults. API.md mandates rate limits; host `.env`/config not verified to apply them. | docs vs config not inspected | M | Verify `config('api.rate_limits')` / `RateLimiter` usage in each app's API bootstrapping; add a CI assertion that rate-limit middleware is attached to auth + public API routes. | 1–2 days |
| SEC-12 | **CRITICAL** | module-organizations-teams, module-roles-permissions, module-identity-core | Foundation modules owning tenancy, authorization, and authentication ship with ZERO tests (`tests/` = `.gitkeep`). Documentation mandates 100% coverage + two-team negative tests. | tests/ directory listings (verified 2026-10-29) | H | Write Pest suites for every foundation module; gate module release CI on `--min=100`; add two-team negative isolation tests to organizations-teams. | 10–20 days |
| SEC-13 | HIGH | module-organizations-teams / ecosystem | Documented `BelongsToTenant` trait (team global scope + team_id stamping) — core tenant-isolation mechanism — NOT found in the tenancy module src. Existence unverified. | src inspection (verified absence) | M | Locate or implement the trait; add architecture test asserting every tenant-owned model uses it; add negative tests proving cross-team access is blocked. | 3–5 days |
| SEC-04u | MEDIUM (updated) | composer-installer | Path-traversal protection VERIFIED good (regex `^[a-z0-9-]+$` + collision detection). Remaining: no release signing; single-point trust. | LiberuInstaller.php full source | H | Add GPG/Sigstore-signed releases; pin installer tag in app lockfiles; add `/modules` ↔ lockfile integrity CI check. | 2 days |

**CRITICAL finding SEC-12 (confirmed 2026-10-29):** the foundation modules that own tenancy (`module-organizations-teams`), authorization (`module-roles-permissions`), and authentication (`module-identity-core`) ship with **ZERO tests** (`tests/` = `.gitkeep`). The documentation mandates 100% owned-PHP coverage and two-team negative isolation tests. This is the single most serious verified finding — the security-critical core of the ecosystem is untested. See [Deep-Dive-Module-Audit.md](Deep-Dive-Module-Audit.md) for the stratified evidence.

The HIGH findings (SEC-02, SEC-05, SEC-13) are the most actionable alongside SEC-12.

---

## 6. UI/UX and Accessibility Assessment

**[CANNOT ASSESS — no running application was deployed or tested.**]

Per the audit rules, I will not invent observed usability defects. The following are **implementation-level risks to validate in staging**:

- Filament 5 + Livewire 4 provide a reasonably accessible admin surface by default, but custom modules/themes may break keyboard navigation, focus order, and ARIA semantics — **must be tested per module**.
- The documented multi-presentation strategy (React/Vue/Nuxt/RN/Flutter adapters) means UX consistency depends on each adapter team following the design system; the `theme-*` packages are the design-system contract and were not inspected.
- No Lighthouse/axe run was performed. **Required staging test plan:** deploy boilerplate + one product app (CRM or genealogy), run Lighthouse + axe-core on core journeys (login, dashboard, create record, list/tenant switch), and verify the documented "loading/empty/success/destructive" states exist.
- Estimated staging-based UI/UX audit effort: **3–5 person-days** per application.

---

## 7. SaaS Readiness Assessment

### 7.1 Tenancy

- **Documented model:** single-database, shared-schema multi-tenancy via Jetstream `Team` + `BelongsToTenant` global scope + policies + team-scoped files/exports + queue context. Reference implementation: genealogy.
- **Verified:** genealogy uses Jetstream teams (no `stancl/tenancy`). CRM uses `stancl/tenancy`. **The tenancy model is not uniform.** This is the single biggest SaaS-readiness risk: a module built for one model may not isolate correctly in the other.
- **Tenant isolation evidence:** global-scope + policy approach is defense-in-depth *if* every tenant-owned model uses the trait and has a policy. The documentation requires two-team negative tests. **Whether every CRM module (300+) actually applies the trait and has negative tests is unverified and is the key thing to prove before launch.**

### 7.2 Billing and customer lifecycle

- **Implemented:** genealogy only — `laravel/cashier ^16`, Stripe, 7-day trial, £2.49/mo or £24.99/yr, auto-creates Stripe Product/Price on first checkout. This is a genuine, if minimal, SaaS billing path.
- **Missing across the portfolio:** subscription plan/entitlement engine, dunning/failed-payment handling, proration, plan changes, account suspension, data export on offboarding, right-to-erasure workflows. The `billing-laravel` app is a *hosting billing* system (cPanel/Plesk), **not** a SaaS subscription engine for the Liberu apps themselves.
- **Verdict:** billing is a per-app bolt-on, not a shared SaaS platform capability. Commercial SaaS requires a shared entitlement/billing module.

### 7.3 Operations and scaling

| Capability | Status | Evidence |
|---|---|---|
| Horizontal scaling | Supported via Octane (RoadRunner) + Horizon + Redis | composer.json deps |
| Queue/cache separation | Configurable; defaults to `database` (dev) | .env.example |
| Observability | Packages available (Horizon, Pulse, Telescope) | deps; no dashboards/alerts/runbooks observed |
| Backup/DR | `spatie/laravel-backup` available; no backup schedule/restore-test evidence | deps only |
| Monitoring/alerting | Not observed | — |
| Environment separation | `.env.example` only; no prod/staging manifest | — |
| Secrets management | `.env` based; no vault/SOPS integration observed | — |

### 7.4 Compliance

- UK-based company; GDPR/UK-GDPR likely applies. No data-processing records, retention policy, or DPIA observed in the inspected docs.
- MIT-licensed open source — commercially permissive, but **no SBOM, licence-scan, or contributor-CLA evidence** was observed for the large dependency tree.
- **[CANNOT ASSESS]** full compliance posture without access to operational policies.

---

## 8. Operational and Deployment Assessment

- **Docker:** `docker-compose.yml` present in inspected apps; `Dockerfile` in boilerplate. `k8s/` directory present in boilerplate. Good signals, but contents not inspected.
- **CI/CD:** GitHub Actions with composer validate/audit, pint, phpstan, tests, codecov. `permissions: contents: read`. **Branch protections and required checks are org settings — not web-visible; unverified.**
- **Release strategy:** `update-repositories.sh` and `scripts/fleet` tooling referenced (accounting README mentions `scripts/fleet status/clone/run`). Suggests a monorepo-like fleet workflow across many repos. No release/rollback runbook observed.
- **Upgrade/rollback:** `composer.lock` pins versions; `/modules` tracked in Git gives reproducible installs. Rollback = revert lockfile + re-deploy. Reasonable, but no canary/blue-green evidence.

---

## 9. Commercial Viability Assessment

| Factor | Assessment |
|---|---|
| Differentiation | Open-source Laravel alternatives to QuickBooks/HubSpot/WHMCS/etc. Genuine niche if depth is real. |
| Open-source model | MIT, self-hostable — reduces lock-in, good for adoption; monetisation via hosting/services (Liberu Hosting/Services exist). |
| Onboarding friction | High — `composer install` + ~50–300 modules + npm + migrations + Redis + DB is expert-only. No turnkey installer/demo observed. |
| Support burden | Potentially very high given module breadth and per-app inconsistency. |
| Website integrity | **Broken GitHub links** to non-existent `liberu-{x}/{x}-laravel` repos undermine the public commercial funnel. |
| Billing | Only genealogy has SaaS billing; the commercial "Liberu" platform is a docs/planning scope. |
| Cost defensibility | Cannot estimate without labour-rate/infra data; effort estimates provided instead. |

**Commercial blocker:** the ecosystem is currently a portfolio of *self-hostable open-source apps*, not a unified SaaS platform. The "Liberu platform" (cross-product orchestration, executive insights, revenue-and-care orchestration) is explicitly a **documentation/planning scope** without a shipped application repository.

---

## 10. Prioritised Remediation Roadmap

Effort in person-days (pd). Calendar duration depends on team size.

### Immediate (0–2 weeks) — containment
| # | Finding | Work | Effort | Owner |
|---|---|---|---|---|
| R1 | SEC-01 | Ship `.env.production.example` with secure defaults + preflight check | 0.5 pd | DevOps |
| R2 | SEC-02 | Pin `twilio/sdk` in CRM; scan all apps for `*` constraints | 0.25 pd | Backend |
| R3 | SEC-10 | Audit legacy `liberu-*` repos for secrets; archive; fix website links | 1 pd | Sec + Web |
| R4 | SEC-09 | Add Dependabot/Renovate; confirm branch protection | 1 pd | DevOps |

### Short-term (1–2 months) — engineering
| # | Finding | Work | Effort | Owner |
|---|---|---|---|---|
| R5 | SEC-05 | Choose ONE tenancy model; migrate CRM; add architecture test forbidding the other; add cross-team negative tests | 5–15 pd | Architect |
| R6 | SEC-06 | Shared authorizer contract + reference policies; prove every tenant-owned model has a policy | 3–5 pd | Backend |
| R7 | Testing | Make module-level coverage gates run in host CI (not just `app/`); sample-verify 5 high-risk modules (files-media, webhooks, api-access, import-export, identity) | 5–10 pd | QA |
| R8 | SEC-03/04 | Set `minimum-stability: stable`; pin installer; signed releases; `/modules` ↔ lockfile integrity check | 2 pd | DevOps |

### Medium-term (3–6 months) — product/SaaS
| # | Finding | Work | Effort | Owner |
|---|---|---|---|---|
| R9 | §7.2 | Build shared entitlement/billing module (Cashier-based, plan/dunning/suspend/offboard) | 15–30 pd | Backend |
| R10 | §7.3 | Observability stack: dashboards, alerts, runbooks, backup schedule + restore tests | 10 pd | SRE |
| R11 | §7.4 | SBOM, licence scan, GDPR retention/DPIA, CLA | 5 pd | Legal/Sec |
| R12 | UI/UX | Staging deploy + Lighthouse/axe on 2 apps; remediate top issues | 10 pd | Frontend |

### Longer-term (6–12 months) — architecture/SaaS
| # | Finding | Work | Effort | Owner |
|---|---|---|---|---|
| R13 | §3.3 | Prove module fleet depth: audit a statistically meaningful sample of the 300+ CRM modules for real implementation vs stubs; retire/merge hollow modules | 20–40 pd | Architect |
| R14 | §9 | Ship the "Liberu platform" cross-product orchestration app (currently docs-only) | 40+ pd | Product |
| R15 | §8 | Multi-region HA, canary/blue-green, rehearsed rollback | 15 pd | SRE |

**Total indicative effort (assumptions: 1 senior Laravel engineer + 1 SRE + part-time architect; no code-gen of estimates):** ~120–160 person-days for short+medium term; longer-term is open-ended.

---

## 11. Test and Validation Plan (proposed — NOT yet performed)

| Tool | Target | Command (proposed) | Status |
|---|---|---|---|
| `composer audit --locked` | each app | `composer audit --locked` | Not run by auditor; configured in CI |
| Larastan/PHPStan lvl 9 | each app + modules | `vendor/bin/phpstan analyse --no-progress` | Not run by auditor; configured in CI |
| Laravel Pint | each app | `vendor/bin/pint --test` | Not run by auditor; configured in CI |
| Pest 5 | each app + module | `vendor/bin/pest --coverage --min=100` (modules), `--min=99` (host app/) | Not run by auditor |
| Semgrep | module fleet | `semgrep --config p/laravel --config p/owasp-top-ten` | **Proposed; not run** |
| `roave/security-advisories` | apps | add as dev dep | **Proposed; not run** |
| OWASP ZAP | staging deploy | `zap-baseline.py -t https://staging...` | **Proposed; requires authorised staging** |
| Lighthouse + axe-core | staging deploy | `lighthouse https://staging...` + `axe` | **Proposed; requires staging** |
| Two-team negative tests | every tenant-aware module | automated Pest tests | **Proposed; mandated by docs, enforcement unverified** |

---

## 12. Production / SaaS Launch Readiness Checklist

- [ ] ONE tenancy model enforced across all apps (SEC-05) — **NOT MET**
- [ ] Every tenant-owned model has a policy + two-team negative test — **UNVERIFIED**
- [ ] Secure production env defaults + preflight (SEC-01) — **NOT MET**
- [ ] All dependencies pinned, no `*` constraints (SEC-02) — **NOT MET**
- [ ] `minimum-stability: stable` for apps (SEC-03) — **NOT MET**
- [ ] Installer pinned + signed releases + `/modules` integrity check (SEC-04) — **NOT MET**
- [ ] Shared billing/entitlement module (§7.2) — **NOT MET (genealogy only)**
- [ ] Observability dashboards/alerts/runbooks (§7.3) — **NOT MET**
- [ ] Backup schedule + tested restore (§7.3) — **NOT MET**
- [ ] SBOM + licence scan + GDPR DPIA (§7.4) — **NOT MET**
- [ ] Staging UI/UX + a11y audit passed (§6) — **NOT MET**
- [ ] Branch protection + required CI checks confirmed (SEC-09) — **UNVERIFIED**
- [ ] Module fleet depth audit (R13) — **NOT MET**
- [ ] Website GitHub links corrected (SEC-10) — **NOT MET**

**0 of 14 items currently verifiably MET.**

---

## 13. Final Decision Framework

**What is the current state of the ecosystem?**
A well-documented, architecturally ambitious open-source Laravel portfolio with strong *intent* but uneven *implementation consistency*. The modular foundation is real and the documentation is a genuine asset, but tenancy, billing, security defaults, and module depth are not yet uniformly proven.

**Five most important verified findings (updated 2026-10-29):**
1. **SEC-12 (CRITICAL):** Foundation modules owning tenancy, authorization, and authentication ship with **ZERO tests** — the security-critical core is untested despite a documented 100% coverage mandate.
2. **ENG-01 (CRITICAL):** ~2,400 public repositories with near-zero community engagement — extreme breadth, unverified depth.
3. **SEC-13 (HIGH):** The documented `BelongsToTenant` trait (core tenant-isolation mechanism) was not found in the tenancy module's source.
4. **SEC-05 (HIGH):** Inconsistent tenancy — `stancl/tenancy` (CRM) vs Jetstream-Team global scopes (genealogy).
5. **SEC-02 (HIGH):** `twilio/sdk: "*"` unbounded constraint + `minimum-stability: dev` supply-chain widening.

**Any Critical/High security risks?**
YES — SEC-12 is a confirmed CRITICAL: the modules responsible for tenant isolation, authorization, and authentication have no automated tests, so tenant isolation and access control are **unproven**. Combined with SEC-13 (BelongsToTenant trait not located) and SEC-05 (divergent tenancy), tenant data isolation is the dominant risk. ENG-01 (2,400 repos) is a CRITICAL sustainability risk.

**Architectural weaknesses affecting multiple applications?**
Yes — tenancy divergence and the thin shared identity/authoriser core mean a weakness in one app's composition can recur or interact across the fleet.

**Is tenant isolation demonstrated?**
Partially. The *mechanism* (global scope + policy) is documented and used in genealogy. It is **not demonstrated** across all modules/apps; CRM uses a different mechanism. Evidence insufficient for a blanket assertion.

**What prevents production/SaaS launch?**
Insecure default hardening, inconsistent tenancy, missing shared billing/entitlement, missing observability/DR, unproven module depth, broken public website funnel.

**What can be fixed quickly vs structurally?**
Quickly (days): env defaults, dependency pinning, website links, Dependabot. Structurally (weeks–months): tenancy unification, shared billing, observability/DR, module-depth audit.

**Estimated remediation effort (assumptions above):** ~120–160 pd for short+medium term; longer-term open-ended.

**Missing information most limiting confidence:** clone-and-run access (to execute tests/static analysis), org settings (branch protection), staging deployment (UI/UX + dynamic security), and per-repo source verification across the 2,400 repositories.

**What the owner should do next:** (1) fix the website funnel and legacy-org hygiene; (2) decide and enforce a single tenancy model; (3) harden production defaults; (4) make module coverage gates run in host CI; (5) commission a staging-based dynamic security + UI/UX audit on boilerplate + one product app.

### Final recommendation: **CONDITIONAL GO** (leaning NO-GO for SaaS until criticals are fixed)

**Tool-executed evidence (2026-10-30):** PHPStan level 9 passes clean on boilerplate; composer-installer passes 32 tests + audit clean; no hardcoded secrets found; but CVE-2026-45070 confirmed in all 4 apps, and the tenancy/authz foundation modules have ZERO behavioral tests (confirmed by running pest). See [Tool-Execution-Results.md](Tool-Execution-Results.md).

The architectural vision, documentation quality, and modern stack justify proceeding **within the assessed scope** — but only after the Immediate and Short-term roadmap items (R1–R8) are addressed and a staging-based dynamic validation (§11) is performed. Proceeding to a paying-customer SaaS launch additionally requires the Medium-term items (R9–R12). The recommendation is **not** GO because there are now confirmed CRITICAL findings (SEC-12: untested security-critical foundation; SEC-14: fail-open tenancy trait; ENG-01: 2,400-repo fleet 35% empty / 80% untested), plus HIGH findings on tenant isolation (SEC-05, SEC-13, SEC-15). The single most important condition for any progression is that the foundation modules (tenancy, authorization, authentication) gain real test suites with two-team negative isolation tests that demonstrably block cross-tenant access. It is **not** NO-GO because the architecture and documentation are sound, the composer-installer is well-engineered, PHPStan level 9 is clean, the genealogy reference modules prove the model can work, and the remediation path is tractable.

---

*End of report. All findings are sourced from public web evidence gathered 2026-10-18. Tool runs cited are CI configurations observed in-repo, not executions by this auditor. This report does not constitute a penetration-test certificate or a warranty of security.*
