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

Test your setup without a real AUR update:

```sh
aur-ai-review --self-test             # benign PKGBUILD, expect VERDICT: SAFE
aur-ai-review --self-test suspicious  # curl|bash installer, expect a block prompt
```

## Requirements

- [paru](https://github.com/Morganamilo/paru) (yay has no equivalent hook)
- One agent CLI with a headless print mode, authenticated:
  - [Kimi Code](https://github.com/MoonshotAI/kimi-code) — `kimi -p` (default)
  - [Claude Code](https://code.claude.com) — `claude -p`
  - [OpenAI Codex](https://github.com/openai/codex) — `codex exec`

## Configuration

| Env var | Default | What it does |
|---|---|---|
| `AUR_REVIEW` | `on` | `AUR_REVIEW=off paru -S foo` skips the review entirely |
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
