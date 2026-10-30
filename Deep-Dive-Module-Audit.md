# Liberu Ecosystem — Stratified Deep-Dive Module Audit

**Date:** 2026-10-29
**Coverage:** ~12 of ~2,400 repositories deep-inspected (~0.5%). This is a risk-based stratified sample, NOT an exhaustive audit. Method: public web/raw file inspection.
**Purpose:** Test the central hypothesis — are the 2,400 repos real implementations or stubs? — and verify the security-critical foundation modules.

## 1. Repository categorisation (from org listing + naming convention)

| Category | Naming pattern | Estimated count | Sample inspected |
|---|---|---|---|
| Applications | `{domain}-laravel` | ~14 | boilerplate, crm, accounting, genealogy (prior) |
| Foundation modules | `module-{foundation}` | ~25 | organizations-teams, roles-permissions, identity-core |
| Foundation presentation | `module-{foundation}-{api\|filament\|livewire}` | ~75 | (referenced) |
| Product domain modules | `module-{crm\|accounting\|genealogy\|...}-{capability}` | hundreds | crm-cpq, crm-customer-data-platform, accounting-copilot, genealogy-people |
| Product presentation adapters | `module-{product}-{capability}-{api\|filament\|livewire\|react\|vue\|nuxt}` | thousands | crm-cpq-filament |
| Contracts / tooling / themes | `*-contracts`, `composer-installer`, `package-testbench`, `theme-*`, `*-scripts` | dozens | composer-installer |
| **Total** | | **~2,400** | **~12** |

## 2. Deep-inspection results by category

### 2.1 Supply-chain tooling: `composer-installer` — REAL, well-engineered [VERIFIED]

- `composer.json`: type `composer-plugin`, version 1.1.0, `plugin-modifies-install-path: true`, class `Liberu\\ComposerInstaller\\Plugin`.
- `src/LiberuInstaller.php` (read in full):
  - Extends `LibraryInstaller`; supports only `liberu-module` and `liberu-theme`.
  - **Path-traversal protection VERIFIED:** name validated against `/^[a-z0-9]+(?:-[a-z0-9]+)*$/` — rejects `/`, `\`, `.`, so traversal/absolute paths cannot reach the target.
  - **Collision detection VERIFIED:** checks both the current Composer run (`$targets` map) AND the working tree (reads existing `/modules/{name}/composer.json` to detect ownership). Throws on mismatch.
  - README claims tests cover path computation, types, name validation, traversal/absolute-path rejection, collision detection.
- **Verdict:** This is the linchpin of the supply chain and it is competently built. SEC-04 path-safety concern is RESOLVED (good); the remaining risk is the trust/signing model (no signature verification beyond Composer VCS trust) and that a single compromise breaks every consumer.

### 2.2 Foundation: `module-organizations-teams` — REAL but UNTESTED [VERIFIED]

- `composer.json`: name `liberusoftware/organizations-teams`, version 1.6.1, requires `laravel/jetstream ^5.5.3`, `spatie/laravel-activitylog ^5.1.3`, `liberusoftware/module-manager ^1.0`. Has factories + autoload-dev tests.
- `module.json`: category `foundation`, capabilities `foundation.organizations` + `foundation.teams`, `default_enabled: true`. Declared features: Teams, Memberships, Invitations, Ownership, Switching, Current-context boundaries.
- `src/`: Actions/ (AcceptInvitation, InviteMember, TransferOwnership), Contracts/, Models/ (Membership, Organization, Team, TeamInvitation), Policies/ (TeamPolicy), Services/ (CurrentTeamResolver), OrganizationsServiceProvider.
- **`tests/`: contains ONLY `.gitkeep` — ZERO test files.** [VERIFIED]
- **BelongsToTenant trait: NOT FOUND** in the inspected src tree. The documented core tenancy-isolation mechanism (`BelongsToTenant` trait adding `team` global scope + `team_id` stamping, per TENANCY.md) was not present in this module's Models/Services. It may reside in another package or may be unimplemented — **unverified, flagged.**
- **Verdict:** Real, thin, tenancy-critical, and ships with NO tests despite the documentation's 100% coverage mandate and the requirement for two-team negative isolation tests.

### 2.3 Foundation: `module-roles-permissions` — REAL but UNTESTED [VERIFIED]

- `composer.json`: version 1.4.3, requires `spatie/laravel-permission ^8.3.2` + `organizations-teams ^1.0`.
- `module.json`: capability `foundation.authorization`, `default_enabled: true`.
- `src/`: Contracts/, Models/ (Permission, Role), Policies/ (RolePolicy), Registry/, Services/, RolesPermissionsServiceProvider.
- **`tests/`: contains ONLY `.gitkeep` — ZERO test files.** [VERIFIED]
- **Verdict:** The authorization foundation has no automated tests. Policy coverage for all tenant-owned models is unproven.

### 2.4 Foundation: `module-identity-core` — REAL but UNTESTED [VERIFIED]

- `tests/`: empty listing (no contents) — ZERO test files. [VERIFIED]
- Owns only "authentication policy, identifier normalisation, security events"; delegates authorisation to host.
- **Verdict:** The authentication foundation ships without tests.

### 2.5 Product: `module-crm-cpq` — THIN / IN-PROGRESS [VERIFIED]

- `composer.json`: compact one-liner, NO `require`, NO `version`, NO `license`, NO `require-dev`, NO tests config. `extra.laravel.providers` registers `CpqServiceProvider`.
- `module.json`: version 1.0.0, empty `requires` `{}`, empty `suggests` `[]`, `default_enabled: true`. Claims features: Configuration, compatibility, price books, discounts, approvals, margins, amendments, renewals.
- `src/`: Actions/ (PriceQuote, SubmitQuote), Models/ (CpqApproval, CpqQuote), Queries/ (CpqQuery), CpqServiceProvider.
- `database/migrations/`: exists (contents not read).
- `tests/`: empty/no contents — ZERO tests.
- No README (404).
- **Verdict:** Not an empty stub — it has real Action/Model/Query classes — but it is a thin in-progress module with no declared dependencies, no tests, no docs. A `composer require` would install code that may fail at runtime due to undeclared illuminate dependencies.

### 2.6 Product: `module-crm-customer-data-platform` — THIN / IN-PROGRESS [VERIFIED]

- Same pattern as CPQ: minimal composer.json (no require/version/license), src/ has Actions/, Models/, Queries/, Services/, ServiceProvider. No README, no tests observed.

### 2.7 Product: `module-accounting-copilot` — THIN / SLIGHTLY MORE MATURE [VERIFIED]

- `composer.json`: version 1.0.0, description, requires `php ^8.5`, `illuminate/database ^13.38.0`, `illuminate/support ^13.38.0`. Better than CPQ (declares deps) but still NO `require-dev`, NO tests config.
- `src/`: Actions/, Models/, Queries/, CopilotServiceProvider. No tests dir observed.

### 2.8 Presentation adapter: `module-crm-cpq-filament` — MINIMAL [VERIFIED]

- `composer.json`: minimal one-liner, no require, no version, registers `CpqFilamentServiceProvider`. Consistent with adapter pattern but no tests/README.

### 2.9 Reference product: `module-genealogy-people` — MOST MATURE [VERIFIED]

- `composer.json`: version 1.1.0, license MIT, requires php/illuminate/genealogy-core, `require-dev` with `package-testbench` + `pest`, autoload-dev tests. Properly structured.
- `src/`: Actions/, Events/, Models/, Support/, Capability.php, PeopleServiceProvider.
- **`tests/`: Feature/ + Unit/ — REAL test directories.** [VERIFIED]
- **Verdict:** The genealogy reference app's modules are the most mature in the fleet, consistent with TENANCY.md naming genealogy the reference implementation.

## 3. Stratified maturity summary

| Stratum | Implementation | Tests | Dependencies declared | Maturity |
|---|---|---|---|---|
| composer-installer (tooling) | Substantive, security-hardened | Claimed in README | Yes | High |
| Foundation modules (teams, roles, identity) | Real but thin | **NONE (.gitkeep)** | Yes | **Low — critical gap** |
| Genealogy reference modules | Real, structured | **Feature + Unit dirs** | Yes | Medium-High |
| CRM / accounting product modules | Real but very thin | **NONE** | Mostly missing | Low / in-progress |
| Presentation adapters | Minimal scaffold | None | Missing | Low |

## 4. New findings from deep-dive

| ID | Sev | Finding | Evidence | Confidence |
|---|---|---|---|---|
| SEC-12 | **CRITICAL** | Foundation modules owning tenancy (organizations-teams), authorization (roles-permissions), and authentication (identity-core) ship with ZERO tests despite documented 100% coverage mandate and two-team negative-test requirement. | tests/ = `.gitkeep` (verified ×3) | H |
| SEC-13 | HIGH | Documented `BelongsToTenant` trait (core tenant-isolation mechanism: team global scope + team_id stamping) was NOT found in the tenancy module's src. Its existence/location is unverified. | src inspection of organizations-teams | M |
| SEC-04 | MEDIUM (updated) | Installer path-traversal protection VERIFIED good (regex + collision check). Remaining risk: no release signing; single-point-of-failure trust. | LiberuInstaller.php full source | H |
| ENG-06 | HIGH | CRM/accounting product modules have no declared composer dependencies in many cases (no `require`), so `composer install` may install packages that fail at runtime. | crm-cpq, crm-customer-data-platform composer.json | H |
| ENG-07 | MEDIUM | Most product modules + presentation adapters have NO README and NO tests. The documented "every module has a complete README" rule is violated fleet-wide. | Multiple repos | H |
| ENG-08 | MEDIUM | Maturity is highly stratified: genealogy reference modules are real+tested; CRM/accounting are thin in-progress; foundation is real but untested. The "single architecture, uniform quality" claim is not borne out. | Stratified sample | H |

## 5. Coverage honesty

- Deep-inspected: ~12 of ~2,400 repos (~0.5%).
- Categories confirmed via naming pattern + sample: tooling, foundation, genealogy product, CRM product, accounting product, presentation adapter.
- NOT inspected: the ~2,388 remaining repos; legacy `liberu-*` orgs; themes; react/vue/nuxt/react-native/flutter adapters; contract packages (beyond names).
- This sample is sufficient to confirm the **pattern** (stratified maturity, untested foundation, thin product modules) but cannot prove the state of every individual repo. A fleet-wide automated scan (clone all, run `composer test`, count tests, measure src LOC) is required for full confirmation — see proposed validation plan in main audit §11.
