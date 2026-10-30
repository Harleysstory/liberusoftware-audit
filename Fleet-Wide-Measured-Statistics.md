# Liberu Ecosystem — Fleet-Wide Measured Audit (Local Filesystem Scan)

**Date:** 2026-10-30
**Method:** Direct filesystem analysis of 2,395 cloned repositories at `C:\Users\harle\liberu-software` using shell tools (`find`, `grep`) and file reads. These are **measured** statistics, not web-sampled estimates. This supersedes the estimates in the web-based audit.

## 1. Fleet-wide measured statistics

| Metric | Count | % of 2,395 | Measurement method |
|---|---|---|---|
| Total repository directories | 2,395 | 100% | `ls -d */ \| wc -l` |
| Repos with `composer.json` | 1,565 | 65% | `find -name composer.json` |
| Repos with `module.json` | 1,510 | 63% | `find -name module.json` |
| Repos with `README.md` | 2,074 | 87% | `find -name README.md` |
| Repos with `src/` PHP files (real implementation) | 1,549 | 65% | `find -path '*/src/*.php'` unique top-level dirs |
| Repos with NO `src/` PHP files (empty/scaffold) | 846 | 35% | derived (2,395 − 1,549) |
| Repos with real PHP test files | 484 | 20% | `find -path '*/tests/*.php'` unique dirs |
| Repos with ZERO tests (no tests dir or .gitkeep only) | 1,911 | 80% | derived |
| Total source PHP files (`src/`) | 14,094 | — | `find -path '*/src/*.php'` |
| Total test PHP files | 642 | — | `find -path '*/tests/*.php'` |
| Source-to-test file ratio | ~22:1 | — | 14,094 / 642 |
| Total migration files | 1,401 | — | `find -path '*/migrations/*.php'` |
| Files using `BelongsToTeam` trait | 88 | — | `grep "use BelongsToTeam"` |
| Files using `$fillable`/`$guarded` | 2,897 | — | `grep "fillable\|guarded"` |
| Repos requiring `stancl/tenancy` | 1 (CRM only) | — | `grep "stancl/tenancy" composer.json` |

## 2. Headline measured findings

### 2.1 One-third of the fleet is empty [VERIFIED]
**846 of 2,395 repositories (35%) contain no PHP source files at all.** These are empty directories — either failed/incomplete clones, or scaffolded-but-never-implemented repos. This is direct, measured confirmation that a large fraction of the "2,400-repo ecosystem" has no implementation.

### 2.2 Four-fifths of the fleet has no tests [VERIFIED]
**1,911 of 2,395 repositories (80%) have zero automated tests.** Only 484 repos (20%) have any PHP test files. The total source-to-test file ratio is ~22:1. The documentation mandates 100% owned-PHP line coverage per package — this is not remotely met across the fleet.

### 2.3 Security-critical foundation modules are untested [VERIFIED]

| Foundation module | src PHP files | test PHP files |
|---|---|---|
| `module-organizations-teams` (tenancy) | 11 | **0** |
| `module-roles-permissions` (authorization) | 9 | **0** |
| `module-identity-core` (authentication) | 8 | **0** |
| `module-files-media` (file uploads) | 6 | **0** |
| `module-api-access` (API tokens/scopes) | 3 | **0** |
| `module-audit` (audit trail) | 14 | 2 |
| `module-settings` | 4 | 5 |
| `module-webhooks` | 11 | 2 |

The five most security-critical modules — tenancy, authorization, authentication, file handling, and API access — have **zero tests**. This is the confirmed CRITICAL finding (SEC-12).

### 2.4 Tenancy isolation is genealogy-specific, not shared [VERIFIED]

The `BelongsToTeam` trait — the actual tenancy isolation mechanism — lives in `module-genealogy-core/src/Concerns/BelongsToTeam.php` under the namespace `Liberu\Genealogy\GenealogyCore\Concerns`. It is **NOT** in the shared foundation (`module-organizations-teams`). Only 88 files across the entire fleet use it. CRM uses `stancl/tenancy` instead (the only repo that does). Other applications have **no demonstrated tenant isolation mechanism at all**.

### 2.5 BelongsToTeam trait security analysis [VERIFIED]

Read in full from `module-genealogy-core/src/Concerns/BelongsToTeam.php`:

**Mechanism:**
- Adds a global scope `team` to every model using the trait.
- On read: if the table has a `team_id` column, filters by `TeamContext::current()`. If no team context (guest): returns `is_public=true` records, or `1=0` (nothing) if no `is_public` column.
- On create: stamps `team_id` from context; **throws `LogicException`** if no team context — good.
- `scopeForTeam()`: bypasses the global scope with `withoutGlobalScope('team')` — an explicit escape hatch.

**Security concerns:**

| Concern | Detail | Severity |
|---|---|---|
| **Fail-open on missing `team_id`** | `if (!hasColumn('team_id')) return;` — if a tenant-owned model's migration omits `team_id`, the global scope silently skips and ALL records are visible to ALL tenants. No architecture test enforces team_id presence. | **HIGH** |
| **`hasColumn()` per query** | Calls `getSchemaBuilder()->hasColumn()` on every single query — a schema-introspection DB call per query. Performance and connection-cost concern at scale. | MEDIUM |
| **`scopeForTeam` bypass** | `withoutGlobalScope('team')` is an escape hatch. If user-controlled input reaches a `forTeam()` call, it enables cross-tenant access (IDOR). Call sites not audited. | MEDIUM |
| **Genealogy-specific namespace** | `Liberu\Genealogy\GenealogyCore\Concerns` — not a shared foundation trait. Other products cannot use it without coupling to the genealogy product. | HIGH (architectural) |
| **Only 88 adopters** | 88 files `use BelongsToTeam` across the fleet. With 1,401 migrations and potentially hundreds of tenant-owned models, coverage is a fraction — many models may lack isolation. | HIGH |
| **Create-time throw is good** | Throws if no team context — prevents orphan records. Positive. | — |
| **Guest `1=0` fallback is safe** | Guests see nothing unless `is_public` exists. Good default. | — |

## 3. New/updated findings from measured scan

| ID | Sev | Finding | Evidence |
|---|---|---|---|
| SEC-12 | **CRITICAL** | Security-critical foundation modules (tenancy, authz, authn, files, API access) have ZERO tests | Measured: tests/ = 0 for 5 of 8 foundation modules |
| SEC-14 | **HIGH** | `BelongsToTeam` trait is **fail-open**: silently skips tenant scoping if `team_id` column absent → cross-tenant data leak | Full source read of BelongsToTeam.php line 20-22 |
| SEC-15 | **HIGH** | Tenancy isolation is genealogy-product-specific (namespace `Liberu\Genealogy\...`), not shared foundation; only 88 files use it; other apps have no isolation mechanism | grep + namespace inspection |
| ENG-09 | **HIGH** | 846 repos (35%) have NO source code — empty scaffolds | Measured: 1,549 of 2,395 have src/ PHP files |
| ENG-10 | **HIGH** | 1,911 repos (80%) have ZERO tests; source:test ratio 22:1 | Measured |
| SEC-05 | HIGH (confirmed) | Only CRM uses `stancl/tenancy`; all others use genealogy's BelongsToTeam or nothing — divergent tenancy confirmed fleet-wide | grep composer.json: 1 match |
| SEC-13 | HIGH (updated) | Documented `BelongsToTenant` trait does not exist; actual trait is `BelongsToTeam` in genealogy-core, not organizations-teams | grep + source read |
| POS-01 | INFORMATIONAL | 2,897 files use `$fillable`/`$guarded` — mass-assignment protection is widely adopted | grep |
| POS-02 | INFORMATIONAL | `composer-installer` path-traversal protection is genuine and well-engineered | full source read |

## 4. What this measured scan does NOT cover (honest limitations)

- Did NOT run `composer install`, `composer audit`, `phpstan`, or `pest` (would require PHP 8.5 + composer environment; not executed).
- Did NOT read every source file line-by-line (14,094 files). Security pattern searches were targeted, not exhaustive.
- Did NOT perform dynamic testing (no app was run).
- `grep` across the full fleet timed out on some queries (e.g., `extends Model`, `DB::raw`); those counts are incomplete.
- The 846 empty repos may include failed/incomplete clones — a small number may have source in unexpected locations. Spot-checks confirmed the pattern but not every empty repo was verified.
- Migration count (1,401) was measured at maxdepth 5; some deeper-nested migrations may be missed.
