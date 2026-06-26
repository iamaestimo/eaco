# Eaco v2.0 — Project Tracker

> Working hub for the modernization effort. Mirrors the four GitHub issues
> (Phases 1–4) but is the **source of truth while we work**; GitHub issues are
> synced from here later (decision: track locally for now).

Last updated: 2026-06-26

---

## 1. What Eaco is (one-liner)

An **Attribute-Based Access Control (ABAC)** *authorization* framework for Ruby:
authorization decisions live in the **database** as per-record ACLs
(`designator → role`), not in code. You ask `actor.can?(:edit, document)` and
fetch `Document.accessible_by(user)` with a single indexed query.

## 2. Positioning (the "why now")

**Authentication ≠ Authorization.** Rails 8's built-in auth generator and Devise
answer *who are you?* (sessions, passwords, 2FA). Eaco answers *what can you do?*
They are **adjacent layers** — Eaco pairs *with* Devise / Rails 8 auth, it does
not compete with them. Rails 8 shipping auth pushes more apps to the
authorization question sooner, which is Eaco's space.

**Eaco's real peers** are Pundit / CanCanCan / Action Policy (all RBAC, code-defined).
Eaco's defensible niche vs them:

| Capability | Pundit / CanCanCan / Action Policy | Eaco |
|---|---|---|
| Model | RBAC (rules in code) | **ABAC (ACLs in data)** |
| Per-record, runtime-editable sharing | bolt-on / hand-rolled | **native** (`doc.grant :reader, :user, 42`) |
| "List everything user X can see" | N+1 / custom scopes | **one indexed hash-key query** |

**The moat = dynamic, per-record, end-user-managed permissions** (Google-Docs-style
sharing, multi-tenant org hierarchies) + efficient `accessible_by`. v2.0 messaging
must commit loudly to this niche instead of pitching a generic "authorization
framework" (which loses a head-to-head with Pundit on simple role apps).

**Honest boundaries (state these in docs):**
- Simple "admins vs users" apps → Pundit/Action Policy are lighter; use them.
- The rising alternative for ABAC/ReBAC is *externalized* authz (OpenFGA/Zanzibar,
  OSO/Cedar). Eaco's pitch vs those: "stay in your Postgres, no extra service,
  idiomatic Ruby." Name this boundary in the README.

## 3. Version targets (decided 2026-06-26)

- **Ruby floor:** 3.2 (matches Rails 8.0/8.1 minimum; lets us drop legacy shims).
- **Ruby ceiling:** 4.0, open-ended. (Ruby 4.0.0 shipped 2025-12-25; evolutionary,
  mostly backward-compatible — Box & ZJIT are experimental. Eaco audited clean
  against all 4.0 removals: no `cgi`, no `ObjectSpace._id2ref`, no leading-pipe
  `open`, no `Ractor`; uses `Set` only via public API.)
- **Rails:** 7.2, 8.0, 8.1.
- **Next release:** `2.0.0.beta1`.

### CI matrix

| Ruby | Rails |
|------|-------|
| 3.2  | 7.2, 8.0 |
| 3.3  | 8.0, 8.1 |
| 3.4  | 8.1 |
| 4.0  | 8.1 |

---

## Phase 1 — Modernization  → milestone `v2.0.0-beta`  (GitHub #1)

### 1.1 Rails 7.x/8.x compatibility
- [x] Railtie uses `ActiveSupport::Reloader.to_prepare` — *already present in `lib/eaco/railtie.rb`*
- [x] **AR adapter now supports Rails 7.0–8.1** — added compat modules `V70`–`V81`
  (`lib/eaco/adapters/active_record/compatibility/`); full suite green on AR
  7.2.3 / 8.0.5 / 8.1.3 under Ruby 4.0. *Was hard-failing "Unsupported Active
  Record version: 81".*
- [ ] Verify modern PG jsonb handling is optimal (works; revisit for GIN-index/perf in Phase 2)
- [ ] Zeitwerk autoloader: validate inside a real Rails app boot (gem's own suite passes; railtie path not exercised by Cucumber)
- [ ] Drop old `cache_classes` reliance in railtie → `enable_reloading`/`eager_load` (still works via alias; deprecation cleanup)
- [x] Remove deprecated Rails 3.x/4.x — *gemfiles + Appraisals trimmed; floor now Rails 7.2*

### 1.2 Ruby 3.x AND 4.0 compatibility  *(reworded — was "Ruby 3.x")*
- [x] **Suite green on Ruby 4.0.5** (local: RSpec 21/0, Cucumber 33/33 on Rails 7.2/8.0/8.1)
- [x] Ruby 3.4+ `Hash#inspect` spacing — specs now derive expectation from Ruby's own output
- [x] Ruby 4.0 Prism `SyntaxError` wording — relaxed feature expectation
- [ ] Audit keyword-argument usage (Ruby 3.0+ separation) — no failures surfaced, but not exhaustively audited
- [ ] Add `# frozen_string_literal: true` across `lib/` *(deferred: mechanical, needs string-mutation check; ~10 files missing it)*
- [ ] Pattern matching where it improves the DSL (optional)
- [x] Test on Ruby 3.2, 3.3, 3.4, 4.0 (CI matrix) — *local validation done on 4.0 only; CI covers the rest*

### 1.3 Dependency updates
- [x] Modernized Cucumber (3.2 → 11.x); rspec/yard current. Bundler dep still to review.
- [ ] Add `spec.required_ruby_version = '>= 3.2'` to gemspec — *DONE in this pass*
- [ ] Remove deprecated gems (coveralls → simplecov+coverage service? TBD)
- [x] Migrate Appraisals → checked-in `gemfiles/` matrix *(Appraisals trimmed; see open decision on full removal)*

### 1.4 CI/CD modernization
- [x] Travis → GitHub Actions *(done earlier)*
- [x] New Ruby/Rails matrix (3.2–4.0 × 7.2–8.1) + fix `master`→`main` triggers
- [ ] Automated gem releases (release workflow / trusted publishing)

---

## Phase 2 — New Features  → milestone `v2.0.0`  (GitHub #2)

> **Reprioritized 2026-06-26:** weight toward **deepening the ABAC moat** over
> **breadth**. Adapters that just add another datastore (esp. MongoDB) are
> deprioritized; work that makes dynamic per-record ABAC + `accessible_by`
> better/faster/easier is promoted.

### Tier 1 — Deepen the moat (PRIORITIZE)
- [ ] **Hierarchical designators** (org → dept → team → user) — core to the multi-tenant/org ABAC story
- [ ] **OAuth/OIDC/JWT-claims designator** — harvest designators from tokens; modern auth integration, complements Devise/Rails 8 auth
- [ ] **PostgreSQL jsonb GIN indexes for `accessible_by`** — makes the killer feature actually scale; this *is* the moat
- [ ] **DX: Rails generator** for common setups — lowers adoption friction
- [ ] **DX: rake tasks for ACL inspection/debugging** + console helpers
- [ ] **DX: actionable error messages**

### Tier 2 — Breadth (DEPRIORITIZE / demand-gated)
- [ ] SQLite JSON1 adapter — *keep-ish:* low cost, lets people try Eaco without Postgres; good for dev/test
- [ ] MySQL/MariaDB JSON adapter — only if real demand
- [ ] Time-based designators (business hours) — good ABAC demo; candidate for plugin/cookbook rather than core
- [ ] IP/geo-location designators — same as above
- [ ] ~~MongoDB adapter~~ — **defer/drop** (was already a "stretch goal"); revisit only on concrete demand

---

## Phase 3 — Documentation & Education  (GitHub #3)

> Frame migration guides as **"graduation from Pundit/CanCanCan"** (your sharing
> needs outgrew RBAC), not "replacement". Lead every doc with the auth-vs-authz
> distinction.

### 3.1 Docs site
- [ ] Choose stack (GitHub Pages + VitePress/Jekyll)
- [ ] Getting-started guide
- [ ] Migration guide from Pundit/CanCanCan
- [ ] Full DSL reference with examples

### 3.2 Tutorials
- [ ] 1: Basic — protect a Document
- [ ] 2: Multi-role (owner/editor/reader)
- [ ] 3: Group-based team permissions
- [ ] 4: Google-Docs-style sharing  *(showcases the moat)*
- [ ] 5: ABAC for multi-tenant SaaS  *(showcases the moat)*
- [ ] 6: Migrating from Pundit to Eaco

### 3.3 Example apps
- [ ] Document management (Google Docs clone)
- [ ] Project management (Basecamp/Asana style)
- [ ] Multi-tenant SaaS with org hierarchy

### 3.4 Integration guides
- [ ] **Devise** integration *(positioning-critical: auth + authz combo)*
- [ ] Hotwire/Turbo patterns
- [ ] API-only Rails
- [ ] Testing authz with RSpec

---

## Phase 4 — Community & Sustainability  (GitHub #4)

### 4.1 Community
- [ ] Announcement blog post (lead with auth-vs-authz + the moat)
- [ ] Talk at a Ruby meetup/conference
- [ ] GitHub Discussions
- [ ] CONTRIBUTING.md + contribution guidelines

### 4.2 Maintenance
- [ ] GitHub Sponsors
- [ ] Documented release process
- [ ] Public roadmap (this file → published)
- [ ] Identify/mentor co-maintainers

---

## Open decisions / questions

- [x] ~~**Upgrade Cucumber 3.2.0 → 9.x?**~~ **DONE (2026-06-26):** upgraded to
  Cucumber 11.1.1. No step-definition/World changes were needed (the gem only
  uses `World do` + `Before do`). Dropped the unmaintained `yard-cucumber`
  plugin, which was the hard blocker (`cucumber < 4`). Banner silenced via
  `CUCUMBER_PUBLISH_QUIET` in CI. Green on Rails 7.2/8.0/8.1, Ruby 4.0.
- [ ] **Full Appraisal removal?** Currently trimmed Appraisals + checked-in gemfiles
  (both kept in sync). Decide whether to drop the `appraisal` dev dep entirely and
  hand-maintain gemfiles, or keep Appraisal as the generator.
- [ ] **Repo home:** issues live on `iamaestimo/eaco`, but gemspec/README still point
  at upstream `ifad/eaco`. Decide if this fork is the maintained home (affects
  gemspec `homepage`, badges, README links).
- [ ] **`coveralls`** is unmaintained-ish; replace with `simplecov` + a current
  coverage service?
- [ ] When to actually bump `VERSION` to `2.0.0.beta1` and cut a release.

## GitHub issue sync notes (apply later when we sync)

- **#1:** rename "1.2 Ruby 3.x Compatibility" → "Ruby 3.x **and 4.0**"; change
  "test with 3.1, 3.2, 3.3" → "3.2, 3.3, 3.4, 4.0"; strengthen "remove 3.x/4.x"
  to "floor = Rails 7.2 / Ruby 3.2".
- **#2:** restructure into Tier 1 (moat) / Tier 2 (breadth) as above; mark MongoDB
  deferred.

## Changelog of this effort

- **2026-06-26:** Created tracker; repositioned README (auth-vs-authz + comparison);
  Phase 1 mechanics — trimmed gemfiles/Appraisals to Rails 7.2/8.0/8.1, added
  Rails 8.0/8.1 gemfiles, rewrote CI matrix (Ruby 3.2–4.0), fixed `master`→`main`
  triggers, added `required_ruby_version` + version bump to `2.0.0.beta1`.
- **2026-06-26 (cont.):** Ran the suite locally on Ruby 4.0.5 + PG 18 and made it
  green on Rails 7.2/8.0/8.1: added AR compat modules `V70`–`V81` (the big one —
  adapter rejected AR > 6.1), added `ostruct` dev dep, fixed Ruby 3.4 `Hash#inspect`
  and Ruby 4.0 Prism `SyntaxError` test expectations, removed `.config/cucumber.yml`
  (Cucumber 3.x ERB-profile crash on Ruby 3.4+). Result: RSpec 21/0, Cucumber 33/33.
