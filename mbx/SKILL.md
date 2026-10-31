---
name: mbx
description: Build, test and diagnose Rust projects with mbx (mr_boxington), a caching wrapper around Cargo. Use this whenever the user mentions mbx or mr_boxington, asks to run cargo commands through mbx, wants to know why a Rust build missed the cache or stayed slow, asks about mbx cache hits, stats, disk use, garbage collection, or a `target` symlink pointing into ~/.cache/mbx, or wants to set mbx up. Not for ordinary cargo requests that never mention mbx or build caching.
---

# mbx

mbx puts a shared compile cache in front of Cargo. Prefix any Cargo command with it:

```sh
mbx build --release
mbx test --workspace
mbx clippy --all-targets -- -D warnings
mbx +1.91 check          # choose a toolchain, named the way rustup names it
```

This needs no setup. `mbx --help` and `mbx <command> --help` are the authority for flags, so check them when something here looks out of date with `mbx --version`.

## What to expect from a run

Each run ends with a summary line, which is the evidence for whether the cache helped:

```
mbx[cache]: 1 hits, 4 misses, 35 not looked up, 3 bypassed; 0 B downloaded, 0 B uploaded, 162.7 MiB stored locally
```

Keep these in mind when reporting results, because each one is easy to misread:

- **The first build of anything has no hits.** "Not looked up" means there was no earlier recording to compare with. It populates the cache and says nothing about whether the cache works.
- **A no-op rebuild is not a cache result.** If Cargo finds everything fresh it finishes in milliseconds with no compilations, so that speed comes from Cargo. Only claim a speedup from `hits` above zero or from `mbx stats`.
- **Profiles have separate entries.** `--release` and debug builds differ in arguments, so switching between them misses. The same applies to changes in `RUSTFLAGS` and features.
- **Some work is never cached**, such as rustdoc and some C compilations. Those show up as "bypassed".
- **Workspace crates being edited can stay uncached** because mbx keeps private incremental state for them.

mbx only changes how compilation happens, so test results mean what they mean under Cargo. A test that skips itself still reports `ok`. If a suite finishes in 0.00s, check for an environment gate before calling it a pass.

## The `target` symlink

With managed targets on (`mbx doctor` reports this), running mbx in a checkout can turn its `target/` into a symlink to `~/.cache/mbx/targets/v1/<hash>`. Paths like `target/release/foo` keep working, but:

- Tell the user when this happens for the first time in a repo, since it is a visible change to their checkout even though `target/` is normally git-ignored.
- Build output lives on the cache disk and counts toward its budgets. mbx can prune it, so don't treat `target/` as durable storage.
- Plain `cargo` follows the same link.

## Plain `cargo` versus `mbx`

Plain `cargo` bypasses mbx unless setup is active. Check with `mbx setup --status` and `mbx doctor`; the doctor's `setup` line says whether `cargo` on `PATH` goes through mbx. When the user asked for mbx, prefix the command rather than changing their configuration.

## Finding out why a build missed

Go from cheapest to most expensive:

| Question | Command | Runs a build? |
|---|---|---|
| Where did the compile time go, and what would remove it? | `mbx analyze` | No |
| What changed for each missed crate? | `mbx explain --last` | No |
| Why did compilations bypass the cache? | `mbx explain build <same args>` | Yes |
| Is the installation healthy? | `mbx doctor` | No |
| What has it saved overall? | `mbx stats` | No |
| How big is the cache? | `mbx cache stats`, `mbx cache largest`, `mbx cache projects` | No |
| What would cleanup remove? | `mbx gc --dry-run` | No |
| What is configured? | `mbx settings ls` | No |

Start with `mbx analyze`, because it ranks causes by time and names the fix. `mbx explain --last` and `mbx analyze` read the newest build recorded for the current workspace, so run them from inside it. `mbx explain <cargo command>` only reports compilations that actually run, so if everything is fresh it will show little.

## Commands that change things

Run these only when the user asks, and say what they affect first. Their effects reach beyond the current repo or are costly to undo.

| Command | Effect |
|---|---|
| `mbx setup` | Installs a Cargo shim. With a mise scope it edits mise config and runs `mise reshim`. It also writes rust-analyzer's user `rust-analyzer.toml` unless that file has its own check settings. `mbx setup --status` is the read-only check, and `--uninstall` undoes it. |
| `mbx settings set`, `unset` | Changes the global config, which affects every project on the machine. |
| `mbx clean [WORKSPACE]` | Removes the managed target, its `target` link and incremental state. Shared cache objects stay, but the next build must restore or rebuild, and binaries under `target/` disappear. |
| `mbx cache remove` | Does the same as clean and also forgets the workspace's cache claims. |
| `mbx gc` | Evicts shared cache objects that other checkouts may use. Run `--dry-run` first. |
| `mbx adopt` | Moves existing `target` directories under mbx management. Use `--dry-run` first, especially with `-r`. |
| `mbx cache import`, `export` | Import adds to the store and can consume a directory export. Export writes a cache export. |

mbx prunes itself (`gc.auto = true` by default), so disk pressure rarely needs a manual `gc`.

Avoid `mbx tui` in non-interactive sessions, because it takes over the terminal. `mbx tui --once` prints a plain snapshot instead.

## Less common commands

- `mbx exec <command>` runs a non-Cargo build, such as CMake, with its C and C++ compilations cached.
- `mbx reserve --cpus 4 --memory 6GiB -- <command>` holds scheduler capacity for heavy work outside mbx, such as a Docker build. `--memory` needs `scheduler.memory` to be set.
- `mbx prefetch build --workspace --release` downloads predicted actions from a remote cache. It needs `remote.url` to be configured.
