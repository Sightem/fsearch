# FSearch

Whole-disk file search for Linux and macOS, with fuzzy names, typo tolerance
and indexed content search. Use the CLI, desktop terminal picker, JSON-lines
API, or Rust crate. Linux measurements and upstream macOS comparisons are
listed separately below.

```
cargo build --release && ./target/release/fsearch install   # -> ~/.local/bin/fsearch
fsearch fsearch main              # find files by name
fsearch 'ext:rs grep:apply_dir'   # search inside files
```

## Linux installation

Requires Rust and Linux 5.17 or newer with fanotify named file-handle support:

```sh
cargo build --release --locked
sudo -v
./target/release/fsearch install --login
fsearch status
```

The installer puts the CLI in `~/.local/bin/fsearch`, a root-owned daemon
binary in `/usr/local/bin/fsearch`, and enables `fsearch-<uid>.service`.
The daemon runs as the installing user with `CAP_SYS_ADMIN` for filesystem
marks and `CAP_DAC_READ_SEARCH` for file-handle resolution. Directory scans
and content reads still check the ordinary user's permissions; the index
and socket are private. No capabilities are assigned to a user-writable binary.

Name indexing crosses local ext4, btrfs, xfs and f2fs mounts without following
symlink loops or duplicate bind trees. Unsupported filesystems and `/proc`,
`/sys`, `/dev`, `/run`, `/tmp`, `/boot`, `/var/cache`, `/var/tmp`, `/var/log`,
`/var/spool` and `/lost+found` are excluded. Content indexing covers eligible
text files in your home, with the existing generated/cache-directory exclusions.

Data lives in `${XDG_DATA_HOME:-$HOME/.local/share}/fsearch`. Linux has no
durable fanotify replay journal, so restarts reconcile the disk while live
events are queued. `fsearch uninstall` removes the service and privileged
binary but preserves the user CLI and index.

The rebased content cache uses `FSCSEG11`; older caches are rebuilt
automatically without changing source files. Filename notifications are
batched at about 100 ms. Content updates wait for two seconds of quiet per
folder, or at most five minutes for a continuously changing folder.

Check an installed daemon with `python3 tests/linux_smoke.py`; use
`sudo -v && python3 tests/linux_smoke.py --restart` to check offline edits too.
Run `python3 demo/linux_bench.py --samples 25` for serial warm-cache name,
typing, sort, content and real picker-reload timings on the installed daemon.
Add `--restart` with cached sudo authorization to measure five warm-disk
service starts and their background reconciliation time.

## Desktop picker

Install the Python 3 helper with `install -m 755 fsearch-desktop ~/.local/bin/`,
then run `fsearch-desktop` in a Nerd Font terminal. It uses the existing `fzf`
(tested with 0.74.4); no Python packages or nnn installation are required.
The desktop entry on this workstation opens Contour with FiraCode Nerd Font.

Results show [nnn-style single-cell icons](https://github.com/jarun/nnn/blob/master/src/icons.h),
local modification dates, human-readable sizes and parent folders. Terminal
width comes from the controlling TTY, including the first fzf reload.

| Key | Action |
|---|---|
| Alt-R / Alt-N / Alt-T | Relevance / name / type |
| Alt-D / Alt-A / Alt-S | Newest / oldest / largest |
| Enter / Ctrl-O | Open selected file / parent folder |
| Ctrl-P / Ctrl-R / Esc | Toggle metadata details / refresh / close |

The search box accepts the same filename filters as the CLI. Filenames with
quotes, tabs and newlines retain their exact paths when opened; control
characters are escaped in the display. Details never render file contents.



## Linux workstation measurements

Arch Linux after rebasing onto upstream `b88f4e9`: 5.12 million files/folders
and 963,516 indexed text files. The earlier baseline had 5.15 million
files/folders and 970,169 text files, so these are repeated workloads on a
slightly changed live corpus, not an identical frozen dataset.
Serial warm-cache requests use 25 samples each; typing uses 250 requests.
Full workload details, memory counters and the preserved `before_rebase`
baseline are in [`demo/linux_benchmark.json`](demo/linux_benchmark.json);
reproduce with [`demo/linux_bench.py`](demo/linux_bench.py).

| Workload | Before p50 | Rebased p50 | Rebased p95 |
|---|---:|---:|---:|
| Global fuzzy filename search | 0.92 ms | 0.99 ms | 1.07 ms |
| Scoped typo search | 0.08 ms | 0.02 ms | 0.03 ms |
| Typing sequence, per request | 2.46 ms | 0.05 ms | 0.35 ms |
| Global newest-first sort, top 200 | 7.92 ms | 7.14 ms | 7.46 ms |
| Global name sort, top 200 | 17.23 ms | 16.37 ms | 34.11 ms |
| Global type sort, top 200 | 17.12 ms | 15.13 ms | 30.21 ms |
| Indexed content search | 23.93 ms | 1.96 ms | 2.45 ms |
| Picker reload including startup, 200 rows | 43.99 ms | 42.17 ms | 45.90 ms |
| Live file creation | 103.79 ms | 102.61 ms | 102.75 ms |

Live rename and deletion also take about 103 ms, measured by 5 ms polling.
All measured content queries completed. The indexed content workload read
15 files instead of 1,981; the new gram filters discard most candidates.
The full picker still includes Python and CLI startup, so faster daemon
queries don't translate into a comparable whole-window speedup.
After warm searches, RSS was 674 MiB: 105 MiB anonymous memory and 568 MiB
file-backed pages. The observed RSS high-water mark was 832 MiB, and the
content cache occupied 1.46 GiB on disk. Restart correctness was checked,
but startup latency wasn't retimed; earlier startup and renderer-only
measurements remain under `before_rebase` rather than being relabeled.

## Upstream macOS speed

M4 Max, 8.3M files on disk, vs the previous version (76d612f). Same
results. Medians.

| | before | now | |
|---|---:|---:|---:|
| find a file by name | 1.0 ms | 0.13 ms | 7.7× |
| typing a whole filename, all keystrokes | 23 ms | 3.0 ms | 7.5× |
| search inside files | 14 ms | 2.3 ms | 6.2× |
| slowest 10% inside files (Chromium) | 38 ms | 2.3 ms | 16× |
| CLI, launch to answer | 5.5 ms | 3.3 ms | 1.7× |
| first run, fully searchable | 101 s | 49 s | 2.1× |
| first run, peak memory | 1.2 GB | 0.9 GB | |
| index on disk (Chromium) | 0.32 GB | 0.76 GB | 2.4× bigger |

The index grew because each file carries a filter that lets search skip
files without the text.

## Queries

```
fsearch 'readme in:~/Developer'          # inside a folder
fsearch 'type:image size:>5mb mtime:<7d'
fsearch 'ext:rs regex:fn\s+\w+_dir'      # regex inside files
fsearch 'sym:apply_dir'                  # where it's defined
```

Words are fuzzy, and 5+ letter words forgive one typo (`mian.rs` finds
`main.rs`). Also `'exact`, `^prefix`, `suffix$` and `!exclude`. Filters:
`ext:` `type:` `kind:` `in:` `size:` `mtime:` `re:` `path:` `grep:` `regex:`
`sym:` `limit:` `sort:`. Content search is smart-case.

Filename sorts are `relevance` (default), `name`, `type`, `modified-desc`,
`modified-asc` and `size-desc`, applied across all matches before `limit:`.
Type puts folders first and groups files by case-insensitive extension.
For example, `fsearch 'ext:pdf sort:modified-desc limit:200'`.

## macOS Full Disk Access

From a terminal with Full Disk Access it indexes everything. As a login
item (`fsearch install --login`), grant `~/.local/bin/fsearch` access in
System Settings > Privacy & Security, again after each rebuild. Without
access it skips protected folders instead of prompting.

## API

JSON lines over `~/Library/Application Support/FSearch/fsearch.sock` on
macOS or `${XDG_DATA_HOME:-$HOME/.local/share}/fsearch/fsearch.sock` on Linux,
or through `fsearch stdio`:

```json
{"q": "fsearch main", "limit": 20}
{"q": "", "sort": "modified-desc", "limit": 200}
{"op": "grep", "pattern": "apply_dir", "in": "~/Developer"}
```

Or link the crate:

```rust
let engine = fsearch::Engine::start(fsearch::Options { dir: fsearch::default_dir(&home), home: home.clone(), skip: None })?;
let hits = engine.search(&fsearch::Query::parse("fsearch main", &home)?)?;
```

An app and the CLI share one index: the first process owns it, the rest
follow.

## How it works

- macOS crawls with `getattrlistbulk` and replays FSEvents on restart.
- Linux crawls with `readdir`/`fstatat`, monitors mounted filesystems with
  fanotify and reconciles after restarts or event loss.
- Names live in one mmap'd file, laid out folder by folder so `in:` is a
  range. A bitmap of each distinct name's letters prunes scoring.
- Content search deduplicates identical files and uses trigram postings plus
  per-content gram filters to avoid unrelated reads. Matches are read fresh
  from disk; nanosecond inode change timestamps detect equal-size edits even
  when modification timestamps are restored.
