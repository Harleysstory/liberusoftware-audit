# Liberu Software Ecosystem — Independent Technical & Commercial Audit

> **Auditor:** Harley Bradford (independent consultant — architecture, Laravel security, SaaS strategy)
> **Audited organisation:** Liberu Software (`github.com/liberusoftware`), part of the Liberu Group, UK
> **Audit dates:** 18–30 October 2026
> **Repository under audit:** ~2,395 public repositories across the `liberusoftware` GitHub organisation
> **Final verdict:** **CONDITIONAL GO** — proceed only after the critical findings below are remediated

---

## 1. What this repository contains

This repository is the complete, evidence-based audit of the Liberu Software ecosystem. Every finding is backed by a direct filesystem measurement of all 2,395 cloned repositories, executed tool runs (composer audit, PHPStan level 9, Pest test suites, secret scans), and source-code review of the security-critical components.

| Document | What it contains | Read it when |
|---|---|---|
| **[Liberu-Ecosystem-Audit.md](Liberu-Ecosystem-Audit.md)** | The main report. All 8 audit phases: discovery, engineering quality, security, UI/UX, SaaS readiness, operations, commercial viability, remediation roadmap, launch checklist, and the final decision framework. | You want the complete picture |
| **[Findings-Register.md](Findings-Register.md)** | Quick-reference register of every finding with severity, confidence, affected component, and remediation effort. | You want to look up a specific finding |
| **[Fleet-Wide-Measured-Statistics.md](Fleet-Wide-Measured-Statistics.md)** | The measured data behind the audit: 2,395 repos scanned for source code, tests, migrations, trait adoption. Includes the full security analysis of the `BelongsToTeam` tenancy trait. | You want the hard numbers and evidence |
| **[Deep-Dive-Module-Audit.md](Deep-Dive-Module-Audit.md)** | Stratified deep-dive: the supply-chain installer, foundation modules, product modules, presentation adapters, and the genealogy reference — each inspected at source level. | You want to know what is real vs. scaffold |
| **[Tool-Execution-Results.md](Tool-Execution-Results.md)** | Every tool run actually executed: `composer audit` (CVEs found), `pest` test runs per repo, `phpstan` level 9, fleet-wide secret scan. Includes commands and raw results. | You want proof the tools were really run |
| **[Liberu-Audit-Deck.pptx](Liberu-Audit-Deck.pptx)** | 18-slide visual presentation of the entire audit: fleet statistics chart, findings register, tenancy before/after, SaaS readiness matrix, remediation roadmap. | You need to present this to stakeholders |

---

## 2. Executive summary for the recipient

### The one-paragraph verdict

Liberu Software has **genuinely excellent architecture documentation** and a modern, well-chosen technology stack (PHP 8.5, Laravel 13, Filament 5, Livewire 4). The `composer-installer` at the heart of the supply chain is well-engineered and properly tested, and PHPStan level 9 passes cleanly on the main application. **However**, the ecosystem cannot responsibly support a commercial SaaS offering today: the security-critical foundation modules (tenant isolation, authorization, authentication) **ship with zero behavioral tests**; the tenant-isolation trait that does exist is **fail-open by design** and lives in a genealogy product module rather than the shared foundation; **35% of the 2,395 repositories contain no source code at all** and **80% contain no tests**. These are tractable problems, but they must be fixed before any paying customer goes live.

### The measured numbers

| Metric | Value | How it was measured |
|---|---|---|
| Total repositories | **2,395** | Filesystem scan of the full clone |
| Repositories with any source code | 1,549 (65%) | `find -path '*/src/*.php'` |
| **Repositories with NO source code** | **846 (35%)** | Derived |
| Repositories with any tests | 484 (20%) | `find -path '*/tests/*.php'` |
| **Repositories with ZERO tests** | **1,911 (80%)** | Derived |
| Total source PHP files | 14,094 | Filesystem scan |
| Total test PHP files | 642 (22:1 source:test ratio) | Filesystem scan |
| Files using the tenancy trait | 88 | `grep "use BelongsToTeam"` |
| Apps using `stancl/tenancy` | 1 (CRM only) | `grep` across all composer.json |
| Confirmed CVEs in main apps | CVE-2026-45070 (symfony/mime) | `composer audit --locked` (executed) |

### The five most important findings

1. **SEC-12 (CRITICAL):** The foundation modules that own tenant isolation, authorization, and authentication have **zero behavioral tests**. Confirmed by running `pest` directly: only shared "boundary" tests from the testbench pass; no domain tests exist. The security-critical core is unproven.
2. **SEC-14 (HIGH):** The `BelongsToTeam` tenancy trait is **fail-open** — if a model's table lacks a `team_id` column, the global scope silently skips, exposing all records to all tenants. No architecture test enforces `team_id` presence.
3. **SEC-15 (HIGH):** Tenant isolation is **genealogy-product-specific**, not a shared foundation capability. Only 88 files use the trait. CRM uses `stancl/tenancy` instead. Other applications have no isolation mechanism at all.
4. **ENG-01 (CRITICAL):** The fleet is inflated to 2,395 repos, of which 846 (35%) are empty and 1,911 (80%) are untested. This is an unsustainable maintenance and supply-chain surface.
5. **SEC-16 (MEDIUM):** **CVE-2026-45070** (symfony/mime email header injection) is present in all four main applications' lockfiles, confirmed by executed `composer audit`. Fix is trivial: `composer update symfony/mime`.

### What is genuinely good

- PHPStan **level 9 passes with zero errors** on the boilerplate application
- The `composer-installer` is secure: **32 passing tests** verify path-traversal rejection and collision detection
- **No hardcoded production credentials** anywhere in the fleet (verified by secret scan)
- Genealogy reference modules have real Feature/ + Unit/ test suites — proof the model works when applied
- Architecture documentation is among the best open-source specs the auditor has reviewed

---

## 3. How to read the findings

Every finding in the register uses this structure:

- **Severity:** CRITICAL (credible path to severe compromise) / HIGH (significant risk) / MEDIUM (meaningful but constrained) / LOW (hardening) / INFORMATIONAL (observation)
- **Confidence:** High / Medium / Low — how certain the auditor is, given the evidence
- **Verified vs. Suspected:** Verified findings were directly observed in source, filesystem measurement, or tool output. Suspected findings are inferred. Nothing in this audit was assumed without evidence.
- **Effort:** Estimated person-days (pd) for remediation, assuming one senior Laravel engineer

**Important honesty note:** The absence of a CRITICAL *network-exploitable* vulnerability is **not** proof of security. No dynamic testing (ZAP, staging deployment) was performed. Tenant isolation is unproven — that is the dominant risk, not any single exploit.

---

## 4. What the recipient must do next (priority order)

### Immediate — this week (containment, ~2.5 person-days total)

1. **Fix CVE-2026-45070** in all four apps: `composer update symfony/mime` (SEC-16, 0.25 pd)
2. **Pin `twilio/sdk`** in CRM (currently `"*"`) and scan all apps for unbounded constraints (SEC-02, 0.25 pd)
3. **Ship secure production env defaults** (`.env.production.example` with `APP_DEBUG=false`, `SESSION_ENCRYPT=true`) and add a deployment preflight check (SEC-01, 0.5 pd)
4. **Audit legacy `liberu-*` orgs** for secrets, archive abandoned repos, and fix the broken website GitHub links (SEC-10, 1 pd)
5. **Add Dependabot/Renovate** dependency automation (SEC-09, 1 pd)

### Short term — next 1–2 months (structural, ~13–32 person-days)

6. **Write real test suites for the foundation modules** — tenancy, authorization, authentication — including two-team negative isolation tests that prove cross-tenant access is blocked (SEC-12, 10–20 pd). **This is the single most important condition for any progression.**
7. **Fix the fail-open trait** or replace it with fail-closed scoping, with an architecture test that fails any tenant-owned model lacking `team_id` (SEC-14, 2 pd)
8. **Move tenant isolation into the shared foundation** and pick ONE tenancy model across all apps (SEC-15/SEC-05, 5–15 pd)
9. **Make module-level coverage gates run in host CI**, not just the ~5 composition classes in `app/` (ENG-02, 2 pd)

### Medium term — 3–6 months (SaaS enablement, ~30–45 person-days)

10. Build a shared billing/entitlement module (currently only genealogy has SaaS billing)
11. Observability stack: dashboards, alerts, runbooks, backup schedule with tested restore
12. SBOM, licence scan, GDPR DPIA
13. Staging deployment + UI/UX and accessibility audit (Lighthouse/axe)

### Longer term — 6–12 months

14. **Consolidate the fleet**: retire or implement the 846 empty repos; merge hollow product modules. A defensible ecosystem is 100–300 deep, tested repositories — not 2,400.
15. Ship the "Liberu platform" cross-product orchestration app (currently documentation-only)

**Full roadmap with dependencies, acceptance criteria, and skill requirements:** see §10 of the main report.

---

## 5. Scope and honest limitations

**What was audited:**
- All 2,395 repositories, measured fleet-wide via filesystem scan (source presence, test presence, migrations, trait adoption, secret patterns)
- Deep source-level review of the security-critical components: `composer-installer`, `module-organizations-teams`, `module-roles-permissions`, `module-identity-core`, the `BelongsToTeam` trait, and all main application composer manifests
- Executed tool runs: `composer install`, `composer audit --locked`, `pest`, `phpstan` (level 9), fleet-wide secret scan
- All architecture documentation (`liberusoftware/documentation`) compared against implementation

**What was NOT done (and is not claimed):**
- No dynamic testing: no application was deployed to staging; no ZAP scan, Lighthouse, or UI/UX testing was performed
- No line-by-line review of all 14,094 source files; security-critical components were read in full, the rest measured structurally
- Dependency vulnerability data reflects `composer audit` at execution time (2026-10-30)
- The audit does not constitute a penetration-test certificate or a warranty of security

---

## 6. Suggested citation

> Bradford, H. (2026). *Liberu Software Ecosystem — Independent Technical & Commercial Audit*. Conducted 18–30 October 2026 against the `liberusoftware` GitHub organisation (2,395 repositories). Method: full-clone filesystem measurement, executed toolchain (PHP 8.5.12, Composer 2.11.0, PHPStan, Pest), and source review of security-critical components. Verdict: CONDITIONAL GO.

---

## 7. Contact and follow-up

For questions about any finding, the evidence behind it, or the remediation plan, open an issue in this repository or contact the auditor directly. If you proceed with remediation, the auditor recommends re-running the validation plan in §11 of the main report after each phase completes, and a follow-up dynamic security assessment (staging deployment + ZAP + Lighthouse) before any paying-customer launch.
