# aur-ai-review

AI code review for AUR packages, wired into [paru](https://github.com/Morganamilo/paru)'s
`PreBuildCommand` hook. Before paru builds any AUR package, an AI agent
(Kimi, Claude, Codex, …) reviews the packaging changes for supply-chain
attacks — and if it flags something, the build stops until you explicitly
allow it.

```
:: Downloading PKGBUILDs...
[aur-ai-review] gsplus-git r556.4805720-4: asking kimi to review (diff)…
  - source URL still points at the official GitHub repo; checksums updated
    to match the new release tarball; no .install script changes.
  VERDICT: SAFE
[aur-ai-review] gsplus-git: SAFE — proceeding.
==> Making package: gsplus-git r556.4805720-4 ...
```

## Why

Everyone knows you should review the PKGBUILD diff paru shows before
building from the AUR. In reality the diffs are long, most updates are
benign version bumps, and the review gets skipped. This makes the review
happen anyway — by an agent that reads the whole thing every time.

## How it works

paru's `PreBuildCommand` (`man paru.conf`) runs a command via `sh -c`
before each AUR build, with the working directory set to the package's
PKGBUILD dir (a git clone) and `PKGBASE` / `VERSION` in the environment.
A non-zero exit aborts the install.

`aur-ai-review`:

1. Diffs `PKGBUILD`, `.SRCINFO`, `*.install`, and patches against the
   last commit it reviewed (tracked in
   `~/.local/state/aur-ai-review/`). First run for a package = full review.
   Unchanged packages skip instantly.
2. Sends the diff to the agent with a supply-chain-focused prompt (moved
   source URLs, `curl | sh`, base64 blobs, `.install` shenanigans,
   `$HOME`/credential access, …) and requires a final verdict line:
   `VERDICT: SAFE | SUSPICIOUS | MALICIOUS`.
3. SAFE → records the commit, build proceeds. Anything else — including
   the agent being unreachable — asks `y/N` on the terminal. No terminal,
   no `y` → the build is refused.

A **hard denylist** runs before the agent, and before `AUR_REVIEW=off`.
If the pkgname is in `~/.config/paru/aur-blocklist` or
`/etc/pacman.d/aur-blocklist`, the build is refused with no prompt.

Every verdict is appended to `~/.local/state/aur-ai-review/reviews.log`.

Agents with tool access (e.g. Kimi) may go beyond the diff and inspect
the downloaded source tarball as well. Note the review covers *packaging*;
for `-git` packages the upstream source moves independently of the AUR
packaging, so a SAFE verdict there means "the packaging is unchanged",
not "upstream is trustworthy".

## Install

```sh
install -Dm755 aur-ai-review ~/.local/bin/aur-ai-review

mkdir -p ~/.config/paru
cat >> ~/.config/paru/paru.conf <<'EOF'
[bin]
PreBuildCommand = /home/USER/.local/bin/aur-ai-review
EOF
```

(Adjust the path; `~` is not expanded in paru.conf.)

Optional — seed the denylist (ships with `hyprland-fixes` from the
2026-08-28 aur-general report):

```sh
install -Dm644 blocklist ~/.config/paru/aur-blocklist
```

Test your setup without a real AUR update:

```sh
aur-ai-review --self-test             # benign PKGBUILD, expect VERDICT: SAFE
aur-ai-review --self-test suspicious  # curl|bash installer, expect a block prompt
aur-ai-review --self-test blocklist   # expect exit 1 (hard refuse)
```

## Pacman hook (blocks `pacman -U` too)

The paru hook only runs on AUR builds. An already-built package
(`pacman -U`, a GUI helper, a cached `.pkg.tar.zst`) never hits it.
The alpm hook refuses Install/Upgrade of any name in
`/etc/pacman.d/aur-blocklist`, regardless of how it got there.

```sh
sudo install -Dm755 contrib/pacman/check-aur-blocklist \
  /usr/local/lib/pacman/check-aur-blocklist
sudo install -Dm644 contrib/pacman/00-aur-blocklist.hook \
  /etc/pacman.d/hooks/00-aur-blocklist.hook
sudo install -Dm644 blocklist /etc/pacman.d/aur-blocklist
```

`IgnorePkg` in `/etc/pacman.conf` / paru.conf only skips `-Syu`. It does
**not** stop an explicit `-S` or `-U`. Keep it as a courtesy; the hook
is the actual block.

Add more names to the overlay files (`aur-blocklist.local`, one pkgname
per line) as reports land. After changing `/etc/pacman.d/aur-blocklist`,
the next transaction picks it up — no daemon reload.

## Periodic refresh

There is **no official Arch API** for malicious AUR packages. The AUR RPC
(`https://aur.archlinux.org/rpc/v5/...`) can look up a package; it has no
malware flag. Arch staff's live artefact is a HedgeDoc pad of campaign
names (currently the June 2026 wave). Community repos add other waves.
None of them include later one-off reports such as `hyprland-fixes`.

`aur-blocklist-sync` pulls those dumps, merges them with a local overlay,
and rewrites the blocklist. Overlay names are never dropped. A fetch that
returns too few names, or shrinks the remote set by more than half, is
refused unless you pass `--force`.

```sh
install -Dm755 aur-blocklist-sync ~/.local/bin/aur-blocklist-sync
install -Dm644 contrib/sync/sources ~/.config/paru/aur-blocklist.sources
install -Dm644 blocklist ~/.config/paru/aur-blocklist.local

aur-blocklist-sync --dry-run
aur-blocklist-sync            # writes ~/.config/paru/aur-blocklist
aur-blocklist-sync --self-test
```

Daily timer (user session; updates the paru denylist):

```sh
mkdir -p ~/.config/systemd/user
cp contrib/systemd/user/aur-blocklist-sync.* ~/.config/systemd/user/
systemctl --user enable --now aur-blocklist-sync.timer
```

To keep the alpm hook in sync, install the system unit (needs root) so
`/etc/pacman.d/aur-blocklist` is rewritten too:

```sh
sudo install -Dm755 aur-blocklist-sync /usr/local/bin/aur-blocklist-sync
sudo install -Dm644 contrib/sync/sources /usr/local/share/aur-ai-review/sources
sudo install -Dm644 blocklist /etc/pacman.d/aur-blocklist.local
sudo install -Dm644 contrib/systemd/aur-blocklist-sync.service \
  /etc/systemd/system/aur-blocklist-sync.service
sudo install -Dm644 contrib/systemd/aur-blocklist-sync.timer \
  /etc/systemd/system/aur-blocklist-sync.timer
sudo systemctl enable --now aur-blocklist-sync.timer
```

Those remote lists are **historical campaign names**. Arch often nukes the
malicious commits and leaves the package. A hit means "this name was
compromised at least once", not "it is malicious today". The overlay is
the place for names you want blocked regardless.

## Requirements

- [paru](https://github.com/Morganamilo/paru) (yay has no equivalent hook)
- One agent CLI with a headless print mode, authenticated:
  - [Kimi Code](https://github.com/MoonshotAI/kimi-code) — `kimi -p` (default)
  - [Claude Code](https://code.claude.com) — `claude -p`
  - [OpenAI Codex](https://github.com/openai/codex) — `codex exec`

## Configuration

| Env var | Default | What it does |
|---|---|---|
| `AUR_REVIEW` | `on` | `AUR_REVIEW=off paru -S foo` skips the **agent** review. The denylist still runs. |
| `AUR_REVIEW_AGENT` | `kimi` | Agent CLI to use (`claude`, `codex`, …) |

The agent binary is resolved from `PATH`, then `~/.kimi-code/bin`,
`~/.local/bin`, and `~/.local/share/mise/shims` — so it keeps working in
update flows that run with a minimal environment. If paru runs as root
(e.g. via a system updater), the script re-execs itself as the invoking
user (`SUDO_USER`, falling back to the clone dir's owner) with the right
`HOME`, so the agent finds its credentials in *your* home, not root's.

## Notes & limitations

- paru also runs `PreBuildCommand` when the build is cached, so unchanged
  packages still get the (instant) skip check.
- The verdict protocol is a convention, not a guarantee — an agent can
  always be talked out of a correct verdict by sufficiently adversarial
  input. Treat SUSPICIOUS/MALICIOUS as "go read the diff yourself".
- Cost: one agent call per changed AUR package, typically 30–60 s.

## License

MIT
