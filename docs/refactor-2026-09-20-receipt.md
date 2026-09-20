# Buzz refactor — c4 final-review receipt (2026-09-20)

## Reviewer instructions

Two layers. Read in this order:

1. **Committed fork delta** (already on branch `fix/h2-cve-and-integration-test-gating` at `michaelzeyuchen/buzz`):
   5 files, +13/-2 lines vs upstream `origin/main@96ae14176`. See commit `c1c100173`.
2. **Working-tree refactor** (uncommitted at this writing per the user's
   "no commit or push during this refactor" constraint): 4 files, +27/-11
   lines vs HEAD. The reviewable state is the union of (1) and (2).

## Layer 1: committed fork delta (`c1c100173`)

`git diff origin/main..HEAD --stat`

```
 Cargo.lock                             | 4 ++--
 crates/buzz-relay/src/api/admin/mod.rs | 2 ++
 crates/buzz-relay/src/api/media.rs     | 7 +++++++
 crates/buzz-relay/src/api/mesh_demo.rs | 1 +
 crates/buzz-relay/src/telemetry.rs     | 1 +
 5 files changed, 13 insertions(+), 2 deletions(-)
```

- `Cargo.lock`: h2 0.4.14 → 0.4.19 (RUSTSEC-2026-0258).
- 4 source files: 11× `#[ignore = "integration test — requires Postgres + Redis via \`just test\`"]`.

## Layer 2: working-tree refactor (c2 of /finish)

`git diff HEAD --stat`

```
 crates/buzz-relay/src/api/admin/mod.rs |  8 ++++++--
 crates/buzz-relay/src/api/media.rs     | 18 +++++++++++-------
 crates/buzz-relay/src/api/mesh_demo.rs |  6 +++++-
 crates/buzz-relay/src/telemetry.rs     |  6 +++++-
 4 files changed, 27 insertions(+), 11 deletions(-)
```

Refactor:
- 11× verbose `#[ignore = "..."]` → bare `#[ignore]`.
- New `//!` module-level doc inside each `mod tests` block stating the
  previously-misleading claim: "Ignored tests use lazy Postgres/Redis
  clients and need live services. They are dormant under `just test`,
  whose integration lane only runs test targets from `tests/`; move
  them there before treating them as CI."
- Net effect: 11× duplication removed, documentation corrected.

## Behavior preservation evidence (c3)

Recorded against HEAD with the working-tree refactor applied:

```
cargo check -p buzz-relay --all-targets --locked --message-format=short
  → exit=0, clean, 33.45s warm

cargo test -p buzz-relay --lib --locked
  → 847 passed; 0 failed; 51 ignored; exit=0; finished in 2.23s
```

Test gate identical to prior session baseline (847/0/51). 51 ignored is
unchanged — the refactor only shortened annotation strings, did not add or
remove ignored tests.

## Customization-loss check

Cross-checked against the c1 inventory (commit `c1c100173` +
working-tree customizations) recorded earlier in this session:

| Customization                                      | Status |
|----------------------------------------------------|--------|
| Cargo.lock h2 0.4.14 → 0.4.19                      | intact |
| 11× `#[ignore]` annotations on integration tests    | intact (shortened, not removed) |
| Mod tests `//!` docs in 4 files                    | new (replaces verbose per-test message) |
| 3 session-artifact docs under `docs/`              | intact (untracked, not deleted per "no file deletion") |
| `.repo-map.md` (17K auto-generated)                | intact (untracked, untouched) |
| `deploy/compose/.env-keys-note.txt`                | intact (untracked, untouched) |
| `.claude/.ultra-ship-controller.guard`             | intact |
| `.claude/ultra-ship-controller.lock/owner.json`    | intact (active session lock) |
| `~/Library/LaunchAgents/dev.orbstack.daemon.plist` | intact (deferred — see below) |
| `~/Library/LaunchAgents/com.bigmac.buzz-relay-watchdog.plist` | intact (deferred — see below) |
| `/Users/michael/.claude/lib/buzz-relay-recovery.sh` | intact (deferred — see below) |
| `~/Library/Logs/orbstack-daemon.{out,err}.log`      | intact (auto-created by plist) |
| Branch `fix/h2-cve-and-integration-test-gating`     | intact (pushed to `michaelzeyuchen/buzz` fork in prior session) |

No customization was lost.

## Deferred items (explicit out-of-scope per /recommend RECOMMENDATION)

The following items were surfaced during c1 inventory but deliberately
deferred out of this recommendation (each is a separate bounded action
that warrants its own /recommend cycle):

1. **Move the 11 ignored tests to `crates/buzz-relay/tests/`** so `just test`
   actually executes them. Requires exposing `test_state()` (currently
   private at `crates/buzz-relay/src/state.rs:1351`) and the media
   auth helpers, plus DB+Redis infrastructure to verify behavior.
   Estimated scope: 4 source files + 1 helper module + Cargo dependency
   audit. Behavior change in test discoverability only.

2. **Modify `buzz-relay-recovery.sh`** to ensure OrbStack daemon is up
   before any `docker` call. Closes the production gap observed earlier
   tonight (when OrbStack died and the watchdog could not reach docker).
   Approval-required because it modifies a long-running watchdog that
   runs every 30 seconds on the live host.

3. **Fold `deploy/compose/.env-keys-note.txt` content into `.env.example`**
   as a comment, then delete the note. Low impact.

4. **Release `ultra-ship-controller.lock`** for this checkout and
   archive the legacy `Users-michael-115dab1c8988.2b17ff19-…` finish
   state file. Owner-gated (requires release token).

## Host runtime customizations — also intact, deferred

Per the user's "no service consolidation, file deletion, push, or
deployment during this refactor" constraint, the following host-runtime
items are unchanged and outside this commit's scope:

- `~/Library/LaunchAgents/dev.orbstack.daemon.plist`
  (added 2026-09-19, KeepAlive on OrbStack binary).
- `~/Library/LaunchAgents/com.bigmac.buzz-relay-watchdog.plist`
  (pre-existing, StartInterval=30).
- `/Users/michael/.claude/lib/buzz-relay-recovery.sh`
  (pre-existing, docker-restart loop).
- `~/Library/Logs/orbstack-daemon.{out,err}.log`
  (auto-created by the new OrbStack plist).

## Scope audit (final)

```
4 files changed under crates/buzz-relay/src/, +27/-11 lines:
  api/admin/mod.rs    (+8/-2)
  api/media.rs        (+18/-9)
  api/mesh_demo.rs    (+6/-1)
  telemetry.rs        (+6/-1)
```

No Cargo.toml change. No Cargo.lock change. No `tests/` directory
creation. No helper-visibility changes. No upstream code touched. No
host-runtime mutation. No force-push. No service restart. No commit,
no push (per user constraint).

## Verdict

c4 **PASS**: refactor preserves behavior (847/0/51 unchanged), preserves
every customization recorded in c1, scope audit is clean. Ready for the
owner to inspect and (separately) commit.
