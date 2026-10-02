# Sajid's maintained fork

This fork keeps a small local compatibility patch while retaining upstream history, authorship, and the MIT license. Base: upstream main `1c1c623da4d9d64cf19723a24ac0a800cd18d387`, based on v0.3.0 with two later documentation commits.

## Branches and differences

- `fix/daemon-account-switch`: upstream-facing implementation, commits `319d1eb` (bounded daemon helper and tests), `dabade5` (CLI integration, diagnostics, behavior tests, command docs), `5baaa35` (relative PATH fallback regression fix), and `0836af1` (Unix executable-permission checks and focused resolver tests). Submitted as [upstream PR #164](https://github.com/Loongphy/codex-auth/pull/164); real Linux account recovery and local installation passed.
- `fork/local-install`: the same implementation plus this delivery note and fork publishing guards. Test CI and packaging metadata checks remain intact; upstream preview/npm/release publication is disabled on this fork until a distribution route is selected.
- The daemon diagnosis and restart idea originated in [codejunkienick's PR #163](https://github.com/Loongphy/codex-auth/pull/163). This independently implements an explicit per-command workflow, responsive detection, bounded calls, and visible operational failure without adding a persisted setting. See [switch behavior](commands/switch.md).
- Sajid's CODEX_HOME contribution #51 and isolated-login scratch-directory fix #120 are already in the selected upstream history. The original `feat/codex-home-override` and `fix/login-scratch-codex-home` branches remain preserved. Their relevant behavior is included upstream; nothing was cherry-picked again. Comparing the local login fix with upstream shows only the subsequently removed comment, not an omitted functional fix.

## Local evidence, 2 October 2026

- Previous alias: `codex-auth-dev` → `/home/mpr/random_arbitrary_nothing/codex-auth/zig-out/bin/codex-auth`, reporting `0.3.0-alpha.8`. The executable is retained for rollback. The alias now selects `codex-auth-local/current/codex-auth`, reporting `0.3.0`.
- Codex shell CLI: `/usr/bin/codex` (npm wrapper), `0.160.0`. Managed daemon: `.codex_apos/packages/app-server-daemon/current/bin/codex`, also `0.160.0`.
- Effective home: `/home/mpr/.codex_apos`; registry schema 4; no explicit credential-store setting or credential environment override was found in the inspected daemon/clients.
- Initially disk and registry selected A while supported daemon `account/read` with `refreshToken: false` reported B. Explicit restart repaired the unchanged-file mismatch. Selecting A without the option kept the same daemon PID; A→B→A explicit switches changed PIDs and reported the expected account identities. Configuration/history checks passed. The attached TUI reconnected automatically without exit/resume; interrupted active-task continuation was not tested.
- Baseline: 435 passed, 5 skipped. Final candidate: 448 passed, 5 skipped. The skips are the existing Linux baseline skips. Windows/macOS ReleaseSafe cross-builds passed; their runtimes have not been tested locally.
- Required isolated `run -- list`, touched-file formatting, and `git diff --check` passed. Boundary checks found the same two pre-existing inline tests in `src/cli/live_tui.zig`; no new inline tests or test-only production exports were introduced.

Validation used an empty/synthetic environment, never live account copies:

```sh
task_root=/tmp/codex-auth-daemon-switch-20261002
repo_path=/home/mpr/random_arbitrary_nothing/codex-auth-daemon-switch
zig_dir=/home/mpr/random_arbitrary_nothing/tools/zig/zig-x86_64-linux-0.16.0
cd "$task_root"
env -i PATH="$zig_dir:/usr/bin:/bin" HOME="$task_root/home" \
  USERPROFILE="$task_root/home" CODEX_HOME="$task_root/codex-home" \
  ZIG_GLOBAL_CACHE_DIR="$task_root/zig-global" ZIG_LOCAL_CACHE_DIR="$task_root/zig-local" \
  CODEX_AUTH_CLI_INTEGRATION_INSTALL_PREFIX="$task_root/integration-bin" \
  zig build --build-file "$repo_path/build.zig" --cache-dir "$task_root/zig-local" test -j2 --summary all
env -i PATH="$zig_dir:/usr/bin:/bin" HOME="$task_root/home" CODEX_HOME="$task_root/codex-home" \
  ZIG_GLOBAL_CACHE_DIR="$task_root/zig-global" ZIG_LOCAL_CACHE_DIR="$task_root/zig-local" \
  zig build --build-file "$repo_path/build.zig" --cache-dir "$task_root/zig-local" run -- list
env -i PATH="$zig_dir:/usr/bin:/bin" HOME="$task_root/home" CODEX_HOME="$task_root/codex-home" \
  ZIG_GLOBAL_CACHE_DIR="$task_root/zig-global" ZIG_LOCAL_CACHE_DIR="$task_root/zig-local" \
  zig build --build-file "$repo_path/build.zig" --cache-dir "$task_root/zig-local" \
  -Doptimize=ReleaseSafe -p "$task_root/native-release"
```

Retained final evidence is at `/home/mpr/random_arbitrary_nothing/codex-auth-local/validation/0836af1c647f`: `baseline.log`, `resolver-review-tests.log`, `resolver-review-smoke.log`, and the `resolver-review-{native,windows,macos}.log` build logs. Recreate the empty home directories before repeating in a new task root. Keep the test helper install prefix isolated from the original checkout's installed binary.

## Candidate, verification, and cutover

Candidate executable:

`/home/mpr/random_arbitrary_nothing/codex-auth-local/builds/0.3.0-sajid-0836af1c647f/codex-auth`

The CLI retains upstream's `0.3.0` version string; this is an untagged fork build, not an official upstream release. Its adjacent `build-manifest.json` records source commit, compiler, optimization, binary SHA256, and the recoverable previous executable. Code comes from `0836af1c647fb94547ebfaea81e854f0b096143f`; the delivery-only branch adds no Zig changes.

Before real switching, make affected sessions idle and create a private, permission-preserving backup outside the source repository of `auth.json`, `accounts/registry.json`, `config.toml`, and account snapshot JSON files. Exclude these private backups from Git. File restoration cannot undo server-side OAuth rotation/revocation. Do not run parallel refreshing copies of the same live credentials.

The first real check should repair the existing mismatch, with no credential difference required:

```sh
CODEX_HOME=/home/mpr/.codex_apos \
  /home/mpr/random_arbitrary_nothing/codex-auth-local/builds/0.3.0-sajid-0836af1c647f/codex-auth \
  switch apostolic --restart-daemon
```

This interrupts the implementing daemon. Run it only in the agreed window from an independent terminal. Reconnect and confirm account A through `/status` or supported `account/read` with `refreshToken: false`. A version response alone is insufficient. Selecting A again **without** the option must not restart. Then switch to valid B and back to A with the explicit option, verifying identity and preserved configuration/history each time. A failed/expired account needs token-error diagnosis rather than repeated refresh attempts.

After acceptance, create `codex-auth-local/current` pointing to the verified version directory and change the existing `.bashrc` alias to:

```sh
alias codex-auth-dev='/home/mpr/random_arbitrary_nothing/codex-auth-local/current/codex-auth'
```

Keep the original checkout's binary. Alias rollback is the previous executable path recorded above; a second unchanged copy is at `/home/mpr/random_arbitrary_nothing/codex-auth-local/rollback/before-daemon-candidate/codex-auth`. Local acceptance and installation completed: `current` selects the candidate directory, and the alias and version were verified in a fresh interactive shell. The private acceptance report and permission-preserving backups remain outside Git. After the user's subsequent switch, read-only daemon inspection confirmed account B.

## Updating

Fetch upstream once, inspect changes affecting auth/daemon coordination, and integrate selectively on a new candidate branch. Keep fork delivery changes separate from upstream-facing commits. Run the existing gates above, build into a new version directory, and repeat real account verification when auth/lifecycle behavior changes. Move the local `current` link only after acceptance; retain its previous target for rollback. Drop redundant compatibility patches through normal history when upstream supplies the behavior. Public versions and distribution remain separate decisions.

Deferred: automatic restart defaults; deliberate live/login/import/remove coordination; OAuth snapshot-refresh changes such as #156; public release/npm namespace work.
