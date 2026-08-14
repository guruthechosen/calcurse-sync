# calcurse-sync

Multi-machine sync for [calcurse](https://calcurse.org/) using git, driven by calcurse's
built-in hook system. No daemon, no server, no third-party service — just the two shell
scripts in `hooks/`.

Built and tested against calcurse 4.8.2 on Arch Linux.

---

## Why git and not ical/CSV

The obvious first instinct is to export with `calcurse -x` (ical) or convert to CSV, sync
the flat file, and import it back. Don't. That pipeline is lossy in three ways:

| Problem | Detail |
|---|---|
| **Drops structure** | Recurrence rules and exception dates have no CSV representation. |
| **Breaks note links** | A todo like `[1]>da39a3ee... Make prod tenant` references `notes/da39a3ee...` by SHA1. CSV has nowhere to put that pointer. |
| **Destroys identity** | Sync needs stable per-item IDs to tell "edited" from "deleted + re-added". Strip the UIDs and every re-import duplicates instead of updating. |

Meanwhile calcurse's *native* files are already the ideal sync payload:

- `apts` and `todo` are **line-oriented plain text**, one item per line — which is exactly
  the shape git's merge algorithm handles best. Edits to different items merge cleanly and
  automatically.
- `notes/` is **content-addressed**: each filename is the SHA1 of the note body. Content-addressed
  files can only ever be added or removed, never modified, so they are structurally incapable
  of conflicting.

So we sync the native files verbatim and let git do the merging. CSV is a fine *export*
target if you want your tasks in a spreadsheet; it is the wrong *transport* for sync.

### What about calcurse-caldav?

calcurse *does* ship real bidirectional sync — `/usr/bin/calcurse-caldav`, installed with the
package. It's the better choice if you need your appointments in a phone calendar app, since
it does proper UID/ETag tracking against a CalDAV server.

It is not used here because this setup only targets personal Linux machines, and CalDAV
requires a server (Nextcloud, Fastmail, self-hosted Radicale). Its Python dependencies
(`httplib2`, plus the deprecated and unpackaged `oauth2client` for Google Calendar) are
not installed on this machine.

The two can coexist: `calcurse-caldav`'s `SyncFilter` setting accepts `cal,todo`, so you can
push appointments over CalDAV while git keeps the todo list.

---

## Architecture

Two repos, deliberately separated:

```
~/proj/calcurse-sync/             ← this repo: the tooling. Shareable, no personal data.
└── hooks/
    ├── pre-load                  canonical source for the hook scripts
    └── post-save

~/.local/share/calcurse/          ← your data repo: keep this one PRIVATE.
├── apts                          appointments and events
├── todo                          todo items
├── notes/                        note bodies, filenames are SHA1 of content
├── .gitignore
└── .sync.log                     hook diagnostics (gitignored)

~/.config/calcurse/hooks/         ← symlinks pointing back into this repo
├── pre-load  -> ~/proj/calcurse-sync/hooks/pre-load
└── post-save -> ~/proj/calcurse-sync/hooks/post-save
```

The split matters: the data repo contains real calendar and todo content and **must stay
private**. The tooling repo contains no personal data and can be shared freely.

The live hooks are **symlinks** into this repo so there is a single source of truth and the
scripts can't drift from what's documented and version-controlled. The trade-off: if this
repo is moved or deleted, sync stops silently (calcurse tolerates a missing hook and just
starts normally). If you'd rather not depend on the path, `cp` the hooks instead of `ln -s`
and accept that you'll need to re-copy them after edits.

---

## How it works

calcurse runs executables named `pre-load`, `post-load`, `pre-save`, and `post-save` from
`<confdir>/hooks/` at the corresponding points in its lifecycle. Two of the four are used:

### `pre-load` — runs before calcurse reads the data files

1. Bails out silently unless the datadir is a git repo with an `origin` remote.
2. Commits any stray uncommitted changes (a crash or an out-of-band edit would otherwise
   block the merge, since git refuses to merge into a dirty tree).
3. `git fetch`, bounded by `timeout 15`.
4. Fast-forwards if possible, otherwise attempts a real merge.
5. Pushes any local commits the previous session failed to send — backgrounded via `setsid`.

### `post-save` — runs after calcurse writes the data files

1. Stages everything and **exits early if the diff is empty** — calcurse fires `post-save` on
   every save including no-op saves, so without this guard the history fills with empty commits.
2. Commits synchronously. This is local, takes milliseconds, and is the part that actually
   protects the data.
3. Pushes **detached and in the background** via `setsid`, so quitting calcurse never blocks
   on the network. A failed push is retried by the next launch's `pre-load`.

Commit messages carry the hostname (`calcurse: todo (thinkpad)`) so history shows which machine
made each change.

### Design constraints these scripts respect

- **Never prevent calcurse from starting.** Every failure path in `pre-load` exits 0 and leaves
  local data untouched. Offline, no remote, no repo — all degrade to "just run calcurse".
- **Never write to the terminal.** Hook output lands on the terminal in the instant before the
  TUI paints, corrupting the display. All git output — stdout *and* stderr — is redirected to
  `.sync.log`. Verified: both hooks emit zero bytes on every path, including conflicts.
- **Never let conflict markers reach the data files.** calcurse cannot parse `<<<<<<<` and would
  mangle or discard the file. See below.

---

## Conflict handling

Git merges non-adjacent line edits automatically, so conflicts only happen when two machines
edit **the same item** without syncing in between.

When that happens, `pre-load` **aborts the merge** rather than writing conflict markers into
`todo` or `apts`. Your local data stays authoritative and fully intact, and the hook appends a
high-priority todo:

```
[1] calcurse-sync: merge conflict, resolve in /home/you/.local/share/calcurse
```

Because `pre-load` runs *before* the data files load, that warning shows up in the TUI on the
very same launch — you find out immediately rather than discovering a silently desynced
calendar weeks later.

### Resolving one

**Close calcurse first.** It holds all data in memory and overwrites the files on save, so a
running instance will clobber your resolution.

```sh
cd ~/.local/share/calcurse
git merge origin/main          # redo the merge, this time keeping the markers
$EDITOR todo                   # resolve by hand; delete the <<<<<<< ======= >>>>>>> lines
git add -A
git commit
git push
```

Then delete the `[1] calcurse-sync: ...` warning line — either in the editor above, or from
inside calcurse afterwards.

---

## Setting up a new machine

### 1. Install calcurse and clone this tooling repo

```sh
sudo pacman -S calcurse                        # or your distro's equivalent
git clone https://github.com/guruthechosen/calcurse-sync.git ~/proj/calcurse-sync
```

### 2. Clone the data repo into the calcurse datadir

The datadir must be empty or nonexistent. **If calcurse has already run on this machine**, it
has its own `apts`/`todo` — back them up first, or you'll lose them:

```sh
mv ~/.local/share/calcurse ~/calcurse-backup    # only if it already exists
git clone git@github.com:<user>/calcurse-data.git ~/.local/share/calcurse
```

Merge anything from the backup by hand afterwards — the files are plain text, so appending
lines is enough.

### 3. Link the hooks

```sh
mkdir -p ~/.config/calcurse/hooks
ln -sf ~/proj/calcurse-sync/hooks/pre-load  ~/.config/calcurse/hooks/pre-load
ln -sf ~/proj/calcurse-sync/hooks/post-save ~/.config/calcurse/hooks/post-save
```

The hooks must be executable. They are committed with the executable bit set, so a fresh
clone is already correct; if you copied them around by other means, `chmod +x` them.

### 4. Verify

Both hooks are safe to run by hand — that's the fastest way to confirm the wiring:

```sh
~/.config/calcurse/hooks/pre-load  ; echo "exit=$?"    # expect exit=0, no output
~/.config/calcurse/hooks/post-save ; echo "exit=$?"    # expect exit=0, no output
git -C ~/.local/share/calcurse status -sb              # expect "## main...origin/main"
```

Any output at all from the hooks is a bug — they are supposed to be silent.

---

## Setting this up from scratch (what was done originally)

For reference, creating the data repo the first time:

```sh
cd ~/.local/share/calcurse
git init -b main
printf '.calcurse.pid\n.sync.log\ncaldav/\n' > .gitignore
git add -A
git commit -m "calcurse: initial import of appointments, todos and notes"
gh repo create calcurse-data --private --source ~/.local/share/calcurse --remote origin --push
```

`.gitignore` covers:

- `.calcurse.pid` — runtime lock, machine-specific, recreated every launch.
- `.sync.log` — local hook diagnostics.
- `caldav/` — `calcurse-caldav`'s sync database, if CalDAV is ever enabled alongside git.

**The repo must be private.** Verify rather than assume:

```sh
gh repo view <user>/calcurse-data --json isPrivate,visibility
```

---

## Troubleshooting

Everything goes to `~/.local/share/calcurse/.sync.log`. A healthy setup logs **nothing** on a
clean no-op — entries only appear when something actually happened.

```sh
tail -20 ~/.local/share/calcurse/.sync.log
```

| Symptom | Cause | Fix |
|---|---|---|
| `fetch failed (offline?)` | No network at launch. | Harmless. Local data is used; the next launch retries and pushes. |
| Changes not appearing on the other machine | Background push didn't land. | `git -C ~/.local/share/calcurse push origin main` — or just relaunch calcurse, `pre-load` retries automatically. |
| `MERGE CONFLICT ... aborted` | Same item edited on two machines. | See [Conflict handling](#conflict-handling). |
| Hooks appear to do nothing | Not executable, or symlink target missing. | `ls -lL ~/.config/calcurse/hooks/` — a broken symlink shows as a dangling target. |
| `error: src refspec main does not match any` | The local branch isn't named `main` — the hooks push to `main` unconditionally. Happens if the datadir was `git init`'d rather than cloned. | `git -C ~/.local/share/calcurse branch -M main`, then push once by hand. |
| Junk on screen at launch | A hook wrote to stdout/stderr. | All git calls in the hooks must redirect `>>"$LOG" 2>&1`. |
| History full of empty commits | The `git diff --cached --quiet` guard in `post-save` was removed. | Restore it. |

### A note on testing changes

Don't test hook edits against live data. Both scripts derive the datadir from
`${XDG_DATA_HOME:-$HOME/.local/share}/calcurse`, so you can point them at throwaway clones:

```sh
git clone <data-repo> /tmp/A/calcurse
git clone <data-repo> /tmp/B/calcurse
echo "[2] test item" >> /tmp/B/calcurse/todo
XDG_DATA_HOME=/tmp/B ~/.config/calcurse/hooks/post-save     # commits and pushes from "B"
XDG_DATA_HOME=/tmp/A ~/.config/calcurse/hooks/pre-load      # "A" pulls it
```

That's how the fast-forward, background-push, and conflict-abort paths were all verified
without touching real todos.

---

## Gotchas

- **calcurse holds everything in memory.** Editing the data files while it's running is
  pointless — it overwrites them on save. Always close it before manual git work.
- **Hook changes take effect on the next launch**, not the running instance.
- Relevant `~/.config/calcurse/conf` settings: `general.autosave=yes` means a save (and thus a
  commit) happens on quit. `general.periodicsave=0` disables timed autosaves — raising it would
  produce a commit every N minutes.
- The hooks assume the branch is named `main`.
- `conf` and `keys` live in `<confdir>`, not `<datadir>`, so they are **not** synced by this
  setup. If you want your calcurse config synced too, that's a separate dotfiles concern.

---

## License

MIT — see [LICENSE](LICENSE).
