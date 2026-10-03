# PRP 026 — Codex as the third registered agent

**Issue:** none filed. Implemented directly on request and kept **local-only**
— no push, no PR — per explicit instruction for this fork's work. File an
issue before any of this goes upstream.
**Status:** implemented and verified end-to-end against the real `codex`
binary (`codex-cli 0.146.0`). Not merged to `main`, not pushed anywhere.
**Depends on:** PRP 023 (`agent.rs`'s registry) and PRP 024 (Cursor, the
second agent) — **read both before touching this code**. This PRP assumes
their vocabulary (`AgentId`, `AgentSpec`, `Launch`, `mints_session_ids`,
`supports_profiles`, decision #10's one-watcher-per-agent-root rule) without
re-explaining it.
**Branch / worktree:** `preview/v2-split-terminal-groups+profile-status-bar`,
checked out at `.claude/worktrees/combined-preview/` — a git worktree
alongside the main checkout (`git worktree list` shows all three). **Work in
this worktree, not the main checkout at the repo root** — the main checkout
is on `feat/profile-aware-status-bar`, which does not have this code.
**Scope:** the third agent, end to end. PRP 024 closed with "the honest test
of 023's registry is whether adding one is smaller than this PRP was" — it
was: one `AgentId` variant, a handful of match arms, one new ~300-line
provider module, one new watcher block. Zero changes to the settings panel,
the `+` picker, the Sessions-tab merge, or any storage-key namespacing — all
of that already generalizes over N agents, confirmed by a full frontend
sweep before writing a line of code.

## Where this branch sits relative to upstream

```
main (v1.14.0, upstream willywg/klaudio-panels)
 └─ feat/split-terminal-groups        (split-pane terminal groups)
     └─ feat/profile-aware-status-bar (bottom usage status bar)
         └─ feat/webview-zoom-hotkeys (native webview zoom)
             └─ preview/v2-split-terminal-groups+profile-status-bar
                 └─ 9f1ef8a  feat(agents): add Codex as a third registered agent  ← this PRP
```

Everything below this PRP's own commit is prior, already-integrated work
(the fork sync from v1.10.0→v1.14.0, plus this fork's own split-terminal and
status-bar features). This PRP only describes `9f1ef8a` and the one
production-build gotcha found while getting the resulting binary into daily
use.

## The problem

Two agents are registered: Claude and Cursor (PRP 024). The user asked for
Codex (`codex` / `codex-cli`, OpenAI's CLI agent) as a third, reusing
whatever of Cursor's integration transfers. A peer research session had
already explored Codex's shape against a stale checkout (no `agent.rs`
existed yet in the tree it was reasoning against); that research is folded
in here, re-measured against the real, current registry, and corrected in
two places where it had guessed wrong (see below).

## What was measured — do not re-derive, re-verify if anything here goes stale

All of this was read directly off `codex-cli 0.146.0` installed on the dev
machine, never inferred from docs. If a future `codex` version is involved,
re-run these checks before trusting the registry entry.

- **`codex --version`** → `codex-cli 0.146.0`. `accepts_version` checks that
  the first non-empty line starts with `"codex-cli "` — see
  `src-tauri/src/agent.rs:206-214`.
- **Resume is a subcommand, not a flag.** `codex --help` has no
  `--resume`; `codex resume [SESSION_ID] [PROMPT]` does the job. A literal
  copy of Cursor's `["--resume", id]` arm would have shipped a broken argv.
  See `agent.rs:279-286` — the comment there calls this out explicitly
  because it's the one spot copy-paste from Cursor breaks silently (no
  compile error, just a `codex` that doesn't understand its own flag).
- **Installed via the standalone installer**, not npm: real binary at
  `~/.codex/packages/standalone/releases/0.146.0-x86_64-unknown-linux-musl/bin/codex`,
  PATH shim at `~/.local/bin/codex` (confirmed with `codex doctor`). That shim
  path is the `installer_candidates` entry. `fallback_candidates` also walks
  the node-version-manager shims (`~/.bun`, `~/.volta`, `~/.asdf`, `~/.nvm`)
  because Codex *also* ships as `@openai/codex` on npm — unlike Cursor, which
  only has the standalone installer.
- **`$CODEX_HOME`** (default `~/.codex`) is a single variable controlling
  both config and session storage — structurally like `CLAUDE_CONFIG_DIR`,
  unlike Cursor's split `CURSOR_CONFIG_DIR`/`CURSOR_DATA_DIR`. This would
  make `supports_profiles(Codex) = true` structurally sound, but it's
  deferred anyway (see below) — `project_env.rs` hardcodes
  `CLAUDE_CONFIG_DIR` resolution and would need generalizing to a per-agent
  variable name first, which is a separate, bigger change than adding an
  agent.
- **Session storage**: `$CODEX_HOME/sessions/YYYY/MM/DD/rollout-<timestamp>-<uuid>.jsonl`
  — date-partitioned, **not** project-partitioned, confirmed against 145 real
  files. `cwd` and the session's own id are both nested inside the first
  line's `payload`, not top-level.
- **`payload.session_id` ≠ `payload.id`** — found empirically, not
  anticipated. A conversation resumed more than once has *multiple* rollout
  files (one per resume), each with its own distinct `payload.id` matching
  that file's filename, but all sharing one `payload.session_id` — the value
  every one of `codex resume`/`archive`/`fork`'s own `--help` text calls "the
  session id". `codex_sessions.rs` keys everything off `session_id` and
  collapses files that share one into a single row (most-recently-modified
  file wins). Using `id` instead would have shown every resume of one
  conversation as its own, separately-resumable row.
- **The first `role: "user"` message in every rollout is synthetic** —
  `<environment_context><cwd>...</cwd>...</environment_context>`, never
  something the human typed. The preview extractor
  (`codex_sessions.rs::first_real_user_message`) skips it explicitly
  (`is_synthetic_environment_context`).
- **`blocked_env(Codex)` — measured, not guessed**, same live-process gate
  PRP 024 used for Cursor: ran
  `codex exec --sandbox read-only --skip-git-repo-check` asking it to run
  `env | cut -d= -f1 | sort` (names only — no value ever reached a
  transcript), diffed against the parent shell's names. Codex's child added
  eight names. Three are this *launching* session's own bookkeeping and are
  blocked: `CODEX_CI`, `CODEX_SANDBOX_NETWORK_DISABLED`,
  `CODEX_THREAD_ID` — the #104 shape, where a freshly spawned `codex` that
  inherits them would read the wrong thread id, CI mode, or sandbox decision
  as its own. Five are left alone on purpose, same reasoning PRP 024 used
  for Cursor's `CURSOR_RIPGREP_PATH`/`NO_COLOR`: `GH_PAGER`, `GIT_PAGER`,
  `LC_ALL`, `LC_CTYPE` point at output formatting, not session identity, and
  `NO_COLOR` is a convention a user may set on purpose — stripping it to
  protect no one would uncolour every agent's TUI launched from inside a
  Codex shell.
- **Not measured, deliberately deferred**: whether Codex's shell tool routes
  commands through the user's login shell the way `cursor-agent`'s does
  (relevant to the `#117` pbcopy-shim-ordering fix in
  `shell_integration.rs`). Flag it if clipboard copy misbehaves from inside a
  Codex tab — not blocking, not yet seen.

## The registry entry (`src-tauri/src/agent.rs`)

| Function | Codex's answer | Where |
|---|---|---|
| `spec` | `display_name: "Codex"`, `bin_name: "codex"`, install hint: `npm i -g @openai/codex` / standalone installer / settings override | `agent.rs:88-95` |
| `installer_candidates` | `~/.local/bin/codex` | `agent.rs:120-124` |
| `fallback_candidates` | `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, plus `~/.bun`, `~/.volta`, `~/.asdf`, every `~/.nvm/versions/node/*/bin` | `agent.rs:163-185` |
| `accepts_version` | first non-empty line starts with `"codex-cli "` | `agent.rs:206-214` |
| `mints_session_ids` | `false` — like Claude. No mint-and-resume path; a session id only exists once the rollout file's first line is written | `agent.rs:241-250` |
| `create_session_argv` | `None` | `agent.rs:256-262` |
| `argv` | `New → []`; `Resume(id) → ["resume", id]` — **subcommand**, not `--resume` | `agent.rs:264-288` |
| `extra_env` | `[]` — no warp-equivalent protocol to unlock | `agent.rs:307-...` |
| `blocked_env` | `["CODEX_CI", "CODEX_SANDBOX_NETWORK_DISABLED", "CODEX_THREAD_ID"]` — measured, see above | `agent.rs:376-394` |
| `enabled_by_default` | `false` | `agent.rs:424-430` |
| `supports_profiles` | `false` (deferred — see above) | `agent.rs:437-460` |
| `list_sessions` | `codex_sessions::list_codex_sessions(project_path)` | `agent.rs:462-468` |
| `watch_root` | `codex_sessions::sessions_root()` → `~/.codex/sessions`, default root only | `agent.rs:470-482` |

`AgentId::Codex` was added to the enum and `AgentId::ALL` (`agent.rs:27,31`);
`as_str()` → `"codex"` (`agent.rs:37`).

## `src-tauri/src/codex_sessions.rs` (new module, ~300 lines)

The one piece with no existing analog to copy wholesale — Claude's provider
is keyed by encoded project path, Cursor's by `md5(cwd)` (and doesn't even
trust the hash, matching on `cwd` instead). Codex's is keyed by **date**,
with `cwd` nested inside the first JSONL line.

- `sessions_root()` → `~/.codex/sessions`.
- `list_rollout_files(root)` walks generically (not hardcoded to 3 levels of
  `YYYY/MM/DD`) for every `rollout-*.jsonl` under it — a future change to the
  partition depth wouldn't silently stop finding files.
- `read_session_meta_line(path)` reads **only the first line**, parses it as
  `session_meta`, pulls `payload.session_id` + `payload.cwd` + the line's own
  `timestamp`. Returns `None` for anything not yet a readable first line
  (file still being written) — same contract as `sessions.rs::read_cwd` has
  for Claude, so the watcher can retry on the next debounce tick.
- `first_real_user_message(path)` scans forward, capped at
  `SCAN_LINES_FOR_PREVIEW = 200` lines, for the first `response_item` with
  `role: "user"` whose content isn't the synthetic
  `<environment_context>` block.
- `updated_at` is the file's **mtime**, not a parsed timestamp — avoids
  reading to EOF on every scan, matching `sessions.rs`'s own
  cheap-first philosophy. `created_at` is the first line's own `timestamp`.
- `scan_sessions_root(root, project_path)` is the project-path-parameterized
  core (mirrors `cursor_sessions.rs`'s `chats_root()`/`scan_chats_root()`
  split) — `list_codex_sessions` is a thin wrapper that resolves the real
  root and delegates, which is also what lets tests exercise the scan logic
  against a `TempDir` instead of the real `~/.codex/sessions`. **If you add a
  test here, call `scan_sessions_root` directly, not `list_codex_sessions`**
  — this tripped up the first draft (three of five tests failed silently
  against an empty real directory until this split was made).
- Session-id collapsing: groups rollout files by `payload.session_id`, keeps
  the most-recently-modified file per group. See "What was measured" above
  for why this exists.
- `session_from_rollout(path)` (public wrapper around
  `session_from_rollout_file`) is reused by `session_watcher.rs` so the
  watcher and the provider share one mapping function.

## `src-tauri/src/session_watcher.rs` — the new block

Explicit, separate per-agent install block (`install_codex`), following the
established pattern of `install_claude`/`install_cursor` rather than a loop
over `AgentId::ALL` — this codebase's "enum dispatch over abstraction"
philosophy applies to the watcher too, per decision #10. Codex is closer to
Claude's shape than Cursor's here: it doesn't mint session ids, so (unlike
Cursor, which never emits `session:new`) Codex **needs** `session:new` on
first sighting of a new rollout file, to feed the same FIFO-correlation path
Claude's tabs already use.

- `CODEX_SEEN` — a separate seen-set from Claude's, seeded at boot
  (`seed_codex_seen`) so pre-existing rollout files don't fire a spurious
  `session:new` on first modification.
- `emit_for_codex_rollout(app, path)` — filters to `is_rollout_file`, maps
  via `codex_sessions::session_from_rollout`, emits `session:new` on first
  sighting (payload carries `agent: "codex"`) then always emits
  `session:meta`.
- `install_codex(app)` — resolves `agent::watch_root(Codex)`, no-ops (logs
  and returns `Ok`) if `~/.codex/sessions` doesn't exist, otherwise installs
  a `notify-debouncer-full` watcher identical in shape to Claude's/Cursor's.
- `install()` now calls `install_cursor`, `install_codex`, `install_claude`
  in that order, each logging independently on failure — one agent's watcher
  failing to install must not block the others.

## Everything else — confirmed zero changes needed

This was the actual test PRP 024 posed, and it passed. Before writing a line
of new code, every one of these was read in full to confirm it already
generalizes over N agents:

- **`pty.rs`** — dispatches `agent::argv`/`extra_env`/`strip_blocked_env`/
  `supports_profiles` generically. Its two literal `agent_id == Cursor`
  checks (hook env, shell integration) just evaluate `false` for Codex,
  exactly as they already do for Claude.
- **`agent_hooks.rs`** — 100% Cursor-specific by design (the `stop`-hook
  completion mechanism). Not touched; Codex's completion story is deferred
  (below), so there's nothing yet for it to plug into.
- **`agent_settings.rs`, `agent_commands.rs`, `lib.rs`'s command list** — all
  already take `agent_id: String`, dispatch through `AgentId::parse` + the
  registry. `lib.rs` got exactly one new line: `pub mod codex_sessions;`.
- **Frontend, everything except `lib/agents.ts`** — `context/agents.tsx`,
  `agent-badge.tsx`, `merge-sessions.ts`, `restore-tabs.ts`,
  `sessions-list.tsx`, `session-watcher.tsx`, `notifications.tsx`,
  `terminal.tsx`, `last-session.ts`, `open-tabs.ts`, `App.tsx`,
  `agent-settings-dialog.tsx` — all iterate `AgentId`/`agents.enabledIds()`
  generically already.

## Frontend — the one real change (`src/lib/agents.ts`)

```ts
export const CODEX = "codex";
export const AGENT_IDS = [CLAUDE, CURSOR, CODEX] as const;
// AGENT_DISPLAY.codex: name "Codex", product "Codex", bin "codex",
// short "Co" (two-letter badge — all three agent names start with "C"),
// badgeClass in emerald (orange = Claude, sky = Cursor).
```

Everything downstream of `AGENT_IDS` (the picker, the Sessions-tab merge, the
settings dialog's `For each={agents.agents()}`, badges) picked this up with
**zero further code changes** — that propagation, for free, is the whole
point of 023/024's design, and this PRP is the proof it still holds at N=3.

Test fixtures touched to add a third agent to existing parametrized cases:
`last-session.test.ts` (had used `"codex"` as its example of an *unknown*
agent — now invalid since Codex is real, swapped for `"opencode"`),
`merge-sessions.test.ts`, `restore-tabs.test.ts`, `notifications.test.ts`,
`session-watcher.test.ts` (each got one new test mirroring their existing
Claude/Cursor cases, asserting a third agent is isolated the same way the
second was from the first).

## What Codex doesn't get in this pass (deferred, same shape as PRP 024's list)

1. **Completion notifications (`session:complete`).** No hook/notify
   integration yet — a Codex tab raises no `needsAttention`, fires no toast.
   Whether Codex's `config.toml` has a `notify`-program mechanism comparable
   to Cursor's `hooks.json` is unmeasured. Same shape as Cursor's `#109`
   (closed by PRP 025) — this is Codex's open equivalent.
2. **Per-project accounts (`CODEX_HOME` profiles).** `supports_profiles` is
   `false`. A `.envrc` setting `CODEX_HOME` reaches the spawned process
   (direnv still applies it) but not Klaudio's bookkeeping — same cost PRP
   024 accepted for Cursor's `CURSOR_CONFIG_DIR`/`CURSOR_DATA_DIR`. Codex's
   single-variable shape (`CODEX_HOME` alone, vs Cursor's split pair) means
   this is actually *more* tractable than Cursor's equivalent once
   `project_env.rs` is generalized — worth doing Codex's profile support
   before Cursor's, if either is picked up.
3. **Conversation preview beyond the first real prompt.** Good enough, same
   honesty note PRP 024 made about Cursor's `title` fallback.
4. **The `#117` pbcopy-shim fix**, only if Codex turns out to need it
   (unmeasured, see above).

## Operational gotcha found while shipping this to daily use (read before rebuilding)

Not part of the Codex feature itself, but directly relevant to continuing
work in this worktree: **don't build a release binary with bare
`cargo build --release`.**

In Tauri v2, whether the compiled app loads the frontend from `devUrl`
(`http://localhost:1420`, the Vite dev server) or from the embedded
`frontendDist` (`dist/`) is decided by the `custom-protocol` Cargo feature,
which the **`tauri` CLI** enables automatically for a production build.
Running `cargo build --release` directly — bypassing the CLI — compiles a
binary that is release-optimized but still wired for `devUrl`. With no dev
server running, the window shows a blank page with "Could not connect to
localhost: Connection refused" instead of the app. This happened once in
this worktree and cost a debugging cycle before the fix was found in the
repo's own `docs/release-flow.md`.

**Correct Linux build**, exactly the three steps `package:linux` documents:

```bash
bun run bridge:prepare -- x86_64-unknown-linux-gnu   # stage the statusline-bridge sidecar
bun run build                                        # fresh dist/ via Vite
bun run tauri build --config src-tauri/tauri.bundle.conf.json
```

The last step is really `bun run package:linux` (`package.json`); running it
in two pieces here only to make the sidecar-prep step explicit. It also
produces `.deb`/`.rpm`/`.AppImage` bundles as a side effect — harmless, and
ignorable if you only care about the raw binary at
`src-tauri/target/release/klaudio-panels` (the path the `.desktop` launcher
in `~/.local/share/applications/` already points at). After a build, kill
any running instance (`pkill -f target/release/klaudio-panels`) before
relaunching, since the old process holds the stale binary mapped.

## Verification already done

- `cargo check && cargo clippy -- -D warnings && cargo test` clean (two
  categories of pre-existing flaky failures under parallel execution —
  `clipboard_history`'s socket test, a couple of `project_env` direnv tests
  — reproduce identically on a pristine `upstream/main` checkout and pass in
  isolation; unrelated to this change).
- `bun run typecheck && bun test` clean.
- End-to-end against the real binary: enabled Codex in the Agents dialog
  (binary auto-discovered at `~/.local/bin/codex`), opened a new session via
  the `+` picker, got a real reply in the real `codex` TUI, watched the
  Sessions tab pick up the live rollout file with the correct preview and a
  "CODEX" badge.
- After the build-pipeline fix above: confirmed the rebuilt
  `target/release/klaudio-panels` loads the real UI (not the
  connection-refused error) and restores the previous workspace correctly.

## Acceptance (all met)

1. With Claude, Cursor and Codex all enabled, `+`/"New session" offers all
   three. ✅
2. Choosing Codex opens a real `codex` TUI in a tab. ✅
3. That session appears in the Sessions tab, merged with Claude's and
   Cursor's by recency, badged "Co". ✅
4. Clicking it resumes via `codex resume <id>` (subcommand). ✅
5. Closing and reopening the project restores the Codex tab as dormant and
   wakes it via the same FIFO/`session:new` correlation Claude uses. ✅
   (structurally guaranteed by `mints_session_ids = false` + the watcher
   block; not separately re-verified across a full app restart in this
   pass — worth a manual pass if picking this up again)
6. A Claude `direnv` failure still shows an inline error on Claude's rows
   only — Codex's (and Cursor's) rows stay listed. ✅ (unchanged code path)
7. The settings panel shows Codex's discovered path as placeholder, accepts
   an override, rejects one that fails its `--version` probe. ✅ (generic
   code path, unchanged)
8. Disabling Codex hides every trace of it; re-enabling restores its
   workspace intact. ✅ (generic code path, unchanged)
9. `cargo check`, `cargo clippy -- -D warnings`, `bun run typecheck`,
   `bun test` all clean. ✅

## What's next — picking up from here

In priority order, for whoever (Codex, included — this is written for an
agent to act on) continues this branch:

1. **Decide what to do with this branch.** It is local-only, three commits
   ahead of anything pushed (zoom hotkeys, the Codex-upstream-sync merges,
   and this PRP's commit), on top of two feature branches
   (`feat/split-terminal-groups`, `feat/profile-aware-status-bar`) that
   themselves have open PRs upstream (`project_pr64_status` memory: PR #78
   has 6 open review items from the upstream maintainer). **Do not push or
   open a PR from this branch without asking first** — this was set up
   local-only on explicit instruction, and the instruction was "every
   action", not "once".
2. **Codex completion notifications** (deferred item #1 above) — the
   natural next slice, mirroring PRP 025's Cursor `stop`-hook work. Needs a
   measurement pass on Codex's `config.toml` notify mechanism first, the
   same way PRP 024/025 measured Cursor's hooks before building against
   them.
3. **Codex profiles** (deferred item #2) — now more tractable than Cursor's
   equivalent precisely because `CODEX_HOME` is one variable; doing this
   first would also unblock generalizing `project_env.rs` in a way Cursor's
   own deferred profiles could then reuse.
4. **The fourth agent** — PRP 024 closed by asking "is adding one smaller
   than this PRP was?" and PRP 026 (this one) is the second data point:
   yes, consistently. `opencode` was the other name on PRP 024's list and
   remains unstarted.
