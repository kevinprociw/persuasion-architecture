# Persuasion Club Platform Roadmap 2026

**Created:** June 2026  
**Sources:** Podcast HQ audit, Persuasion Game audit, Site Admin audit, platform comparison  
**Audience:** Solo developer operating three production systems

---

## Guiding Principles

| Principle | What it means in practice |
|-----------|---------------------------|
| **Sunday show reliability** | Podcast HQ is the critical path. No structural changes in the 72 hours before a live show. Game and site changes must not destabilize production runtime. |
| **Evolutionary architecture** | Extract, don't rewrite. Add contracts before moving boundaries. Preserve existing URLs, APIs, and operator workflows. |
| **No rewrites** | No framework migrations, no microservices split, no database-first redesign. Incremental modules, snapshots, and read-only feeds only. |
| **Solo developer practicality** | One deploy per week when possible. Rehearse before Sunday. Prefer scripts, smoke tests, and drop-folder contracts over new infrastructure. |

### System roles (fixed)

| System | Role | Production URL |
|--------|------|----------------|
| **Podcast HQ** | Production runtime — rundown, segments, stage output | `podcast.persuasion.club` |
| **Persuasion Game** | Experience runtime — quiz, audience vote, broadcast game clients | `control.`, `output.`, `host.`, `vote.persuasion.club` |
| **Site Admin** | Publishing / control center — homepage, library, autopost, HQ dashboard | Admin UI via platform |

**Critical coupling today:** Site Admin UI calls `/api/*` on Persuasion Game (`site_cms_api.py` ~12k LOC). Podcast HQ is data-isolated. The platform is three repos with overlapping concepts and no shared IDs.

---

## Current State (Audit Summary)

### Podcast HQ — highest Sunday risk

| Rank | Risk | Impact |
|------|------|--------|
| 1 | Ephemeral in-memory live state | Restart mid-show wipes program, clocks, segments |
| 2 | WebSocket sync with no reconnect | Network blip → control/output desync; manual refresh required |
| 3 | Monolithic `app.py` (~1,866 lines) | Surgical fixes have wide blast radius |
| 4 | No automated tests | Deploy confidence relies on manual rehearsal |
| 5 | Runtime data outside Git | Show defs and uploads need separate backup ritual |

**Sprint 1 (Output Authority): ~32% complete.** Output page and command vocabulary exist; `command_id`/ACK contract and modular backend are missing.

**Safe before Sunday:** Show Builder hash-route fix, runtime backup script, disconnect banner, smoke checklist.  
**Defer past show:** State persistence, monolith split, fat `STATE_SYNC` slimming.

### Persuasion Game — experience runtime, accidental publishing host

| Strength | Gap |
|----------|-----|
| View model layer, game template engine, audience provider bridge | In-memory-only game state |
| Production-ready control/output/host/audience clients | CMS API (~12k LOC) lives in wrong repo |
| 162 library pages, autopost, share/OG engine | Quadruple repo duplication (embedded bundles drift) |
| Operator recovery UX (socket-drop banner) | No auth on control/CMS; no CI; pytest not in deps |

**Verdict:** Operationally usable for live game segments; needs operational hardening, not architectural replacement.

### Site Admin — publishing UI without its own backend

| Strength | Gap |
|----------|-----|
| 162-entry persuasion library, AI Framework API Fetch | 9 MB monolithic JSON state |
| Autopost command center, media library, hub catalog | No backend in repo — depends on TheGame API |
| HQ dashboard shell (BBB-1046) | Static placeholders — no live feeds from Podcast HQ or Game |
| Graceful API → packaged JSON → localStorage fallback | Episode/clip data manually duplicated from Podcast HQ |

### Platform — duplication without integration

| Overlap | Today | Target |
|---------|-------|--------|
| Episodes | Podcast HQ `show_defs/` vs Site `episodes[]` | Publish snapshots from Podcast HQ |
| Media | Separate upload roots + duplicate patterns | Cross-reference registry, not physical merge |
| Frameworks | Library slugs vs segment notes (no linkage) | Optional `framework_slug` on segments/rounds |
| Game segment | `output.persuasion.club` iframe + `vote.persuasion.club` + CMS promo | Read-only status feed to HQ dashboard |
| CMS API | Hosted in Persuasion Game | Gradually extract to Site Admin |

---

## Next 30 Days (June–July 2026)

**Theme:** Protect Sunday. Stabilize prep. Establish contracts.

Focus: Podcast HQ reliability first; parallel low-risk platform hygiene.

### Podcast HQ (priority 1)

| # | Work item | Effort | Risk | Benefit |
|---|-----------|--------|------|---------|
| 1 | **Runtime backup script** — tar `~/podcast_tool_runtime_data/` before deploy | 2–3 hrs | Very low | Recoverable show defs and uploads |
| 2 | **Show Builder hash-route fix** — call `loadData()` when entering `#shows` | 2–4 hrs | Low | Empty dropdown footgun eliminated |
| 3 | **WebSocket disconnect banner** — visible "Disconnected" on control + output | 4–6 hrs | Low | Operator knows when stale; faster recovery |
| 4 | **Deploy smoke script** — `curl` checks for `/`, `/api/admin/shows`, `/output` | 2–3 hrs | Very low | Catch broken deploy before show prep |
| 5 | **Sunday runbook** — no restarts during show, backup path, refresh protocol | 1–2 hrs | None | Reduces panic decisions on show day |
| 6 | Pin `python-docx` in `requirements.txt` | 30 min | Very low | Docx import won't break on fresh venv |

**Show-week rule:** Deploy only items 1, 3, 4, 5 if rehearsed. Freeze `apply_command()` and state model changes.

### Persuasion Game

| # | Work item | Effort | Risk | Benefit |
|---|-----------|--------|------|---------|
| 7 | **Declare canonical paths** — document root-only edits; fix pytest collection | 4–6 hrs | Low | Stops fixes landing in duplicate bundles |
| 8 | Add `pytest` to `requirements.txt`; `pytest.ini` ignores duplicate test dirs | 1–2 hrs | Very low | Runnable test suite |
| 9 | **Game session snapshot** — write `data/runtime/session_snapshot.json` on transition | 1–2 days | Medium | Game segment survives restart |
| 10 | Audio architecture one-pager — output plays show audio; `/audio` manages assets | 2 hrs | None | Operator clarity |

**Do not** split `site_cms_api.py` or delete bundle copies in this window.

### Site Admin

| # | Work item | Effort | Risk | Benefit |
|---|-----------|--------|------|---------|
| 11 | Create `data/platform-status.json` schema (static seed) | 2–3 hrs | Very low | Contract for HQ dashboard |
| 12 | **HQ dashboard reads `platform-status.json`** instead of hardcoded placeholders | 4–6 hrs | Low | Dashboard becomes extensible |
| 13 | Document manual episode/clip curation gap vs Podcast HQ | 1 hr | None | Sets expectation until publish pipeline |

### Cross-platform (contract layer)

| # | Work item | Effort | Risk | Benefit |
|---|-----------|--------|------|---------|
| 14 | Create `site-admin/data/publish-events/` drop folder | 1 hr | None | Future Podcast HQ → Site publish path |
| 15 | **Podcast HQ export endpoint** — `GET /api/exports/show-summary` (read-only metadata) | 4–6 hrs | Low | First integration point; no schema rewrite |
| 16 | Shell script: aggregate show-summary + game `/api/state` → `platform-status.json` | 4–6 hrs | Low | HQ dashboard shows real next-show + game state |

### 30-day success criteria

- [ ] Show Builder loads on any `/admin` entry within 3 seconds
- [ ] Runtime backup exists before every `git pull` on Podcast HQ
- [ ] Disconnect visible on control/output within 2 seconds of drop
- [ ] Post-deploy smoke passes on all three systems
- [ ] `platform-status.json` populated by script (even if manual cron)
- [ ] Zero deploys of command-path or SessionManager structural changes in show week

---

## Next 90 Days (July–September 2026)

**Theme:** Guardrails, thin tests, first extractions, read-only integration.

### Podcast HQ

| # | Work item | Effort | Risk | Benefit |
|---|-----------|--------|------|---------|
| 1 | **pytest smoke suite** — show defs, admin API, one `apply_command`, WS round-trip | 1–2 days | Low | Deploy confidence |
| 2 | Harden `deploy.sh` — optional pre-backup, smoke after restart, non-zero exit on failure | 3–4 hrs | Low | Safer solo deploys |
| 3 | Extract `show_defs.py` from `app.py` (I/O only) | 4–6 hrs | Low–medium | First monolith cut; testable |
| 4 | Log corrupt show-def skips (replace silent `continue`) | 1–2 hrs | Very low | Explains missing shows |
| 5 | **WebSocket auto-reconnect** — backoff + resync on open; rehearse before Sunday | 1 day | Medium | Survives brief blips |
| 6 | **`command_id` + ACK accepted** — additive WS messages; no playback path change | 1 day | Low | Sprint 1 leverage; visible command lifecycle |
| 7 | Extend deploy smoke to cover WebSocket ACK round-trip | 4 hrs | Low | Protects Sprint 1 work |

**Schedule 5–7 in non-show weeks only.**

### Persuasion Game

| # | Work item | Effort | Risk | Benefit |
|---|-----------|--------|------|---------|
| 8 | Archive or remove duplicate bundles (`content/qp_platform/`, etc.) | 1 day | Low | Eliminates drift (P0 debt) |
| 9 | **`CONTROL_API_KEY`** on WS connect (control role) + CMS mutating routes | 1 day | Medium | Closes open operator surface |
| 10 | `systemd` unit + `deploy.sh` for main app (mirror Podcast HQ pattern) | 1 day | Low | Repeatable deploys |
| 11 | Health endpoint — WS connection counts, active round, last command time | 4–6 hrs | Low | Post-show debugging |
| 12 | Extract `AudienceService` from SessionManager (first god-object cut) | 3–5 days | Medium | Safer lifeline changes |
| 13 | `GET /api/exports/game-status` for Site Admin HQ dashboard | 4–6 hrs | Low | Live game state without CMS coupling |

### Site Admin

| # | Work item | Effort | Risk | Benefit |
|---|-----------|--------|------|---------|
| 14 | HQ dashboard: live countdown from `platform-status.json` | 1 day | Low | Real producer cockpit |
| 15 | **"Show Publish" panel** (read-only) — lists `publish-events/` queue | 2–3 days | Low | Review path before merge |
| 16 | Split `x_manual_imports` staging out of `site-content.json` (data hygiene) | 1–2 days | Medium | Smaller saves, faster admin |
| 17 | Add optional `show_def_id` field to `episodes[]` (no migration) | 2 hrs | Very low | Cross-link when publish pipeline lands |

### Cross-platform

| # | Work item | Effort | Risk | Benefit |
|---|-----------|--------|------|---------|
| 18 | **Podcast HQ "Export to Publish Queue"** — writes metadata snapshot to `publish-events/` | 2–3 days | Low | One-way publish; Podcast HQ stays source of truth |
| 19 | Optional `framework_slug` on Podcast HQ segments | 2–4 hrs | Very low | Links show content to library |
| 20 | **`media-registry.json`** — cross-reference URLs across systems (manual/script populated) | 1–2 days | Low | Visibility without merged upload roots |
| 21 | Shared `production.js` — prod hosts, `apiBaseUrl`, `escapeHtml` (copy pattern, not new package) | 1 day | Low | Reduces client duplication |

### 90-day success criteria

- [ ] Command tests pass before any `apply_command` edit ships
- [ ] Game tests runnable with one command; duplicate bundles archived
- [ ] Podcast HQ publish snapshot lands in queue; Site Admin can review it
- [ ] HQ dashboard shows next show title + game session state
- [ ] All three systems have `deploy.sh` + smoke checks
- [ ] Operator auth enabled on game control (env-gated, rehearsed)

---

## Next 12 Months (Through June 2027)

**Theme:** Durable state, clarified boundaries, unified operator experience — still no rewrites.

Organized by platform phase (from platform comparison) mapped to calendar quarters.

### Q3 2026 (Jul–Sep) — Contract layer complete

*Covered in 90-day plan above.*

Deliverables:
- `publish-events/`, `platform-status.json`, `media-registry.json` operational
- Read-only export endpoints on Podcast HQ and Persuasion Game
- HQ dashboard consuming live feeds

### Q4 2026 (Oct–Dec) — Resilience and extraction

#### Podcast HQ

| Work item | Theme |
|-----------|-------|
| Live state snapshot persistence (debounced `show_state_snapshot.json`; env-flagged restore) | Addresses #1 architectural risk |
| Extract `command_router.py` — one command family per PR, behind tests | Maintainability |
| Extract `asset_loader.py` — isolate Media Library scan | Safer media changes |
| Command regression suite — top 20 live commands | Sunday-critical path protection |
| Client-side asset catalog cache in `STATE_SYNC` | Performance as library grows |
| Structured logging → journald (command type, WS connect/disconnect) | Post-show forensics |

#### Persuasion Game

| Work item | Theme |
|-----------|-------|
| Extract `LifelineService` from SessionManager | Second god-object cut |
| Split `site_cms_api.py` into routers (`cms_content`, `cms_autopost`, `cms_frameworks`, `cms_assets`) — same endpoints | Prepare API boundary shift |
| Second game template (`_starter_template`) — validate registry | Proves engine without rewrite |
| Client view-model-only rendering (feature-flag per screen) | Reduces raw-state drift |
| First real audience provider adapter (manual webhook or simple ingest) | External vote source |

#### Site Admin

| Work item | Theme |
|-----------|-------|
| Show Publish merge workflow — approved snapshots → `coming_up`, `episodes[]` | Eliminates dual episode authoring |
| Split `library-content.json` writes from social/history blobs | Faster saves |
| HQ launcher cards — deep links to Podcast HQ control, game control, output, vote | Unified entry point (Phase 4) |

### Q1 2027 (Jan–Mar) — API boundary clarification (Phase 3)

| Route prefix | Current host | Target host | Method |
|--------------|--------------|-------------|--------|
| `/api/site/*`, `/api/autopost/*`, `/api/email/*`, `/api/hub/*`, `/api/frameworks/fetch` | Persuasion Game | Site Admin | Extract router; TheGame `include_router` shim |
| `/api/audience/*`, `/ws/*`, game routes | Persuasion Game | Persuasion Game | Unchanged |
| `/api/admin/shows/*`, Podcast HQ WS | Podcast HQ | Podcast HQ | Unchanged |

**Constraint:** Zero API contract changes for Site Admin UI during transition. Nginx routing change only when shim is proven.

#### Additional work

| System | Item |
|--------|------|
| Podcast HQ | Split `STATE_SYNC` into live vs `CATALOG_SYNC` channels |
| Podcast HQ | `EVENT play_started` / `play_ended` correlated to `command_id` |
| Persuasion Game | SQLite additive layer for vote audit + CMS revision history (not a rewrite) |
| Cross-platform | Unified `/api/content/{type}/{id}` read API (rounds, library items, segment packs) |

### Q2 2027 (Apr–Jun) — Platform maturity

| Work item | System | Notes |
|-----------|--------|-------|
| Event log + derived state (append-only command log) | Podcast HQ | Optional; enables audit trail and replay |
| Native ES modules for clients (no framework migration) | All | `esbuild` optional; no React |
| Typed contracts — Pydantic commands + `STATE_SYNC` schema | Podcast HQ + Game | Reduces client/server drift |
| Segment template type in `game_templates` | Persuasion Game | Podcast rundown steps alongside quiz games |
| Episode processing workspace (`#processing`, `#clips`) | Podcast HQ | Product feature; same asset storage |
| Production hardening — reverse proxy auth, rate limits on upload/proxy | All | If internet exposure grows |

### 12-month target architecture

```
Podcast HQ ──publish-snapshot──► publish-events/
       │                                    │
       └──iframe──► output.persuasion.club  ▼
                              Site Admin ──► persuasion.club
Persuasion Game ──game-status──► platform-status.json
       │
       └── vote / control / host (unchanged)

Shared: slugs, media-registry.json, platform-status.json (files, not a new service)
```

### Explicit non-goals (12 months)

- Multi-instance horizontal scaling
- Cloud migration or Kubernetes
- Framework rewrite (React, Next.js, etc.)
- Microservices split
- Shared database across all three systems
- Merging Podcast HQ and Persuasion Game repos

---

## Sunday Show Operations (Standing Policy)

### 72-hour freeze

No deploys touching:
- `apply_command()` / Podcast HQ command paths
- `SessionManager` command handling
- WebSocket protocol changes (except rehearsed disconnect banner)
- Site Admin content merges that affect homepage live-show sections

### Pre-show checklist (every week)

1. Run `scripts/backup_runtime.sh` on Podcast HQ
2. Run smoke script on Podcast HQ after any deploy
3. Open Show Builder — confirm dropdown populated on `/admin#shows`
4. Load Sunday show def; verify segments and assets
5. Open control + output; confirm WebSocket connected
6. If game segment: confirm `output.persuasion.club` iframe loads; vote URL live
7. Check `platform-status.json` for correct show date

### Mid-show recovery

| Symptom | Action |
|---------|--------|
| Control/output desync | Refresh both tabs; do not restart `podcast-tool` |
| WebSocket disconnect banner | Click reconnect or refresh; verify program state |
| Output blank after accidental restart | Reload show from admin; accept cold start until snapshots ship (Q4) |
| Game iframe dead | Open `output.persuasion.club` directly; game control separate from podcast control |

### Post-show

- `journalctl -u podcast-tool --since "today"` — note anomalies
- File publish snapshot if episode metadata is final
- Update `episodes[]` or queue merge for homepage (manual until Q4 workflow)

---

## Priority Matrix (Full Year)

| Order | Item | System | Why |
|-------|------|--------|-----|
| 1 | Runtime backup ritual | Podcast HQ | Zero code risk; saves show defs |
| 2 | Show Builder hash fix | Podcast HQ | Unblocks weekly prep |
| 3 | Disconnect visibility | Podcast HQ | Live-show awareness |
| 4 | `platform-status.json` contract | Cross-platform | Enables HQ dashboard without coupling |
| 5 | Smoke tests | Podcast HQ | Unlocks safe iteration |
| 6 | Duplicate bundle cleanup | Persuasion Game | Stops drift damage |
| 7 | Game session snapshot | Persuasion Game | Game segment resilience |
| 8 | `command_id` + ACK | Podcast HQ | Sprint 1; low-risk additive |
| 9 | Publish snapshot export | Podcast HQ → Site Admin | Ends dual episode authoring |
| 10 | State snapshot persistence | Podcast HQ | Addresses #1 architectural risk |
| 11 | CMS API extract to Site Admin | Site Admin | Correct ownership boundary |
| 12 | `site_cms_api.py` router split | Persuasion Game | Maintainability before extract |
| 13 | Event log + replay | Podcast HQ | Long-term audit/recovery |
| 14 | Typed contracts | Cross-platform | When client/server drift becomes painful |

---

## Success Metrics

| Metric | 30 days | 90 days | 12 months |
|--------|---------|---------|-----------|
| Show Builder load | Any `/admin` route ≤ 3s | Same | Same |
| Deploy safety | Smoke + backup | + command tests | + snapshot restore rehearsal |
| Live disconnect detection | ≤ 2s banner | Auto-reconnect | + correlated ACK timeline |
| Restart recovery | Manual refresh | Game snapshot restore | Podcast HQ snapshot restore |
| Episode authoring | Manual dual entry | Publish queue review | Approved merge workflow |
| HQ dashboard | Script-fed status | Live countdown + game state | Launcher + publish panel |
| Integration | `show-summary` export | Publish queue | `framework_slug` links across systems |
| Test coverage | Smoke scripts | pytest on both runtimes | Command regression suite |

---

## What We Are Not Doing

| Temptation | Why not |
|------------|---------|
| Merge the three repos | Different runtimes, different failure modes, different deploy cadence |
| Rewrite Site Admin in a framework | 162 library entries and autopost work today; structural surgery is Q4+ |
| Replace FastAPI / vanilla JS | Sunk cost is an asset; team of one knows the stack |
| Single shared media upload root | Production assets ≠ published assets; registry beats merge |
| Big-bang Sprint 1 completion | Output Authority ships incrementally via ACK contract, not iframe surgery before Sunday |

---

## Bottom Line

**Next 30 days:** Protect Sunday. Backup, visibility, smoke checks, and the first read-only integration contract (`platform-status.json`).  
**Next 90 days:** Tests, thin extractions, publish queue, operator auth, and auto-reconnect — all rehearsed in non-show weeks.  
**Next 12 months:** Podcast HQ gets durable state and modular commands; Persuasion Game sheds duplicate bundles and CMS API ownership; Site Admin becomes the real publishing center with a live HQ dashboard — connected by files and slugs, not a platform rewrite.

The dominant risk today is **a single in-memory process with fragile WebSocket clients on show day**. The dominant opportunity is **connecting three mature silos with one-way snapshots and shared identifiers** — without changing what already works on Sunday.
