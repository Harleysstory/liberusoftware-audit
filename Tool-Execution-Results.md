# Liberu Ecosystem — Tool Execution Results (Real Runs)

**Date:** 2026-10-30
**Environment:** PHP 8.5.12, Composer 2.11.0, Node 22.17.0, Git 2.50, Windows 10
**Tools installed:** scoop → PHP 8.5.12 (extensions: mbstring, openssl, curl, gd, zip, pdo_sqlite, pdo_mysql, bcmath, intl, fileinfo), Composer 2.11.0
**Repos under test:** `C:\Users\harle\liberu-software` (2,395 cloned repos)

These are **actually executed** tool results, not CI-configuration observations or web-sampled estimates.

---

## 1. composer-installer (supply-chain linchpin)

| Check | Command | Result |
|---|---|---|
| `composer install` | `composer install --no-interaction` | ✅ Success (78 packages) |
| `composer audit` | `composer audit` | ✅ **No security vulnerability advisories found** |
| `pest` tests | `vendor/bin/pest` | ✅ **32 passed** (33 assertions) — name validation, traversal/absolute-path rejection, collision detection |

**Verdict:** The supply-chain installer is genuinely well-engineered and well-tested. SEC-04 path-safety concern is fully resolved by source + test evidence.

## 2. module-organizations-teams (tenancy foundation)

| Check | Command | Result |
|---|---|---|
| `composer install` | `composer install --no-interaction` | ✅ Success (157 packages) |
| `pest` tests | `vendor/bin/pest` | 5 passed, 2 skipped (59 assertions) — **all from shared package-testbench boundary tests; ZERO module-owned behavioral tests** |
| `phpstan` | (no phpstan.neon) | Not configured at module level |
| `composer audit` | `composer audit` | ✅ No advisories |

**Verdict:** Boundary/architecture tests exist (from testbench) verifying package structure, service provider registration, no host-app coupling, no Filament in domain. But **NO tests for actual tenancy behavior** (team creation, membership, invitation, ownership transfer, cross-tenant isolation). SEC-12 confirmed with test runner evidence.

## 3. module-roles-permissions (authorization foundation)

| Check | Command | Result |
|---|---|---|
| `composer install` | `composer install --no-interaction` | ✅ Success (106+ packages) |
| `pest` tests | `vendor/bin/pest` | 5 passed, 2 skipped (49 assertions) — **all from testbench boundary tests; ZERO module-owned behavioral tests** |
| `composer audit` | `composer audit --format=json` | ✅ No advisories |

**Verdict:** Same pattern as organizations-teams. Architecture tests pass but authorization logic (role assignment, permission checking, policy enforcement, cache invalidation) is untested. SEC-12 confirmed.

## 4. boilerplate-laravel (main application)

| Check | Command | Result |
|---|---|---|
| `composer install` | `composer install --ignore-platform-reqs` | ✅ Success (170+ packages) |
| `composer audit` | `composer audit --locked` | ⚠️ **2 advisories** (see below) |
| `phpstan` | `vendor/bin/phpstan analyse` | ✅ **No errors** (level 9) |
| `pest` tests | `vendor/bin/pest` | 199 passed, **6 failed**, 12 skipped (681 assertions) |

### 4.1 Composer audit advisories (boilerplate — also in crm, accounting, genealogy)

| Package | Advisory | CVE | Severity | Type |
|---|---|---|---|---|
| `symfony/mime` | Email Header Injection via Non-Token Characters in Mime Parameter Names | **CVE-2026-45070** | MEDIUM | Production dependency |
| `phpunit/phpunit` | Code execution by other local users via shared temp directory in PHPT code coverage | GHSA-9vw2-f4vp-x7gf | LOW (dev-only) | Dev dependency |

**Note:** CVE-2026-45070 affects `symfony/mime <8.1.10` — a transitive dependency of Laravel. This is a real, current vulnerability in all four main applications. Fix: update to `symfony/mime >=8.1.10` (or run `composer update symfony/mime`).

### 4.2 Pest test failures

6 failures in `ThemeManagerTest` and `ThemeServiceProviderTest` — all theme view-path resolution issues. The tests expect forward-slash paths (`themes/base/resources/views/...`) but Windows resolves with backslashes. These appear to be **test portability bugs** (Windows path separator), not application defects. The 199 passing tests cover the core application composition, module management, identity, and infrastructure.

### 4.3 PHPStan result

Level 9 analysis passed with **zero errors** — the boilerplate's code is statically clean at the highest practical level. This is a strong positive quality signal.

## 5. Fleet-wide secret scan

| Pattern | Repos scanned | Result |
|---|---|---|
| AWS access keys (AKIA...) | All non-vendor PHP/env/json | **None found** |
| Stripe live/test keys (sk_live_/sk_test_) | All non-vendor | Only in **test files** using fake keys (`sk_test_example`, `sk_live_TOPSECRET`) to verify secrets are encrypted/not exposed |
| GitHub tokens (ghp_...) | All non-vendor | **None found** |
| Slack tokens (xox...) | All non-vendor | **None found** |
| Hardcoded passwords in PHP source | All non-vendor, non-test | **None found** |

**Verdict:** POSITIVE — no hardcoded production credentials discovered. Test files deliberately use fake keys to verify secret-handling safety (e.g., `EncryptedSecretsTest` confirms secrets are encrypted at rest; `TelemetryTest` confirms secrets aren't logged). This is good security practice.

## 6. Summary of real tool execution findings

| Finding | Tool | Status | ID |
|---|---|---|---|
| Supply-chain installer is secure & tested | pest (32 pass) + audit (clean) | ✅ POSITIVE | SEC-04 resolved |
| Tenancy module has no behavioral tests | pest (5 boundary-only) | ❌ CONFIRMED | SEC-12 |
| Authz module has no behavioral tests | pest (5 boundary-only) | ❌ CONFIRMED | SEC-12 |
| Boilerplate PHPStan level 9 clean | phpstan (0 errors) | ✅ POSITIVE | — |
| CVE-2026-45070 in all 4 apps | composer audit | ⚠️ NEW VULN | SEC-16 |
| Boilerplate 6 test failures (Windows paths) | pest | ⚠️ LOW | ENG-11 |
| No hardcoded secrets in source | grep scan | ✅ POSITIVE | POS-03 |

## 7. New finding from tool execution

| ID | Sev | Finding | Evidence |
|---|---|---|---|
| SEC-16 | MEDIUM | CVE-2026-45070 (symfony/mime email header injection) present in all 4 main applications' composer.lock | `composer audit --locked` output (executed 2026-10-30) |
| ENG-11 | LOW | 6 boilerplate test failures on Windows due to path-separator assumptions in theme tests | `vendor/bin/pest` output (executed) |
| POS-03 | INFO | No hardcoded production credentials found; test files use fake keys to verify safe secret handling | fleet-wide grep scan (executed) |
