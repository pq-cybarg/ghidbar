# tools/identity — multi-identity GitHub tooling

Two small tools that prevent the class of mistakes that happen when one
machine has several GitHub identities (work account, personal account,
isolated-purpose account, etc.).

## Why it exists

GitHub identifies SSH-authenticated users by key fingerprint. The
`User <name>` field in `~/.ssh/config` is cosmetic and does **not**
change which GitHub account a push lands under. Multi-identity hosts
where every block uses `Host github.com` will silently auth every
operation as whichever identity comes first in the file — without any
warning to the user.

`ghid` forces explicit identity-aliased URLs
(`git@github-<identity>:owner/repo.git`) and provides safety nets so
the wrong identity cannot slip through. `ghidbar` is a macOS menu-bar
companion that surfaces the current repo's bound identity at a glance.

## Components

| File | What it is |
|---|---|
| `ghid` | Bash CLI: list / doctor / current / init / switch / lock / unlock / guard / verify / whoami / new / rotate |
| `ghidbar.py` | macOS menu-bar app (uses [rumps](https://github.com/jaredks/rumps)) |
| `ghidbar` | shell launcher for the menu-bar app |
| `ghidbar.plist` | launchd template for auto-start at login (uses `__HOME__` placeholder) |
| `install.sh` | One-shot installer / upgrader / uninstaller |

## Install

```bash
# From the repo root:
./install.sh                # ghid + ghidbar in ~/.local
./install.sh --launchd      # also register auto-start at login

# Uninstall:
./install.sh --uninstall    # removes binaries + venv + state
                                           # ~/.ssh/config + keys untouched
```

After install, add `~/.local/bin` to your `PATH` if it isn't already,
then run `ghid doctor` to sanity-check your `~/.ssh/config`.

## Quick start

```bash
# 1. Add an identity (creates an SSH key + alias in ~/.ssh/config).
#    Reuses an existing ~/.ssh/<name>-GitHub key if present.
ghid new myalias

# 2. In a repo, bind it to that identity and lock so any push under a
#    different identity is refused:
cd ~/some/repo
ghid switch myalias
ghid lock myalias

# 3. Confirm GitHub actually resolves the key to the expected user:
ghid verify myalias
# → ✓ alias github-myalias resolves to GitHub user myalias
```

In the menu bar, ghidbar shows `🔑 myalias 🔒` while you're in the repo
(its watched repo is set via the `ghidbar-here` / `gbh` shell function
added to `~/.zshrc` by the install).

## Fresh-repo init (no global git config pollution)

`ghid init` creates a new repo bound to a specific identity *without*
touching `~/.gitconfig`:

```bash
mkdir ~/newproject && cd ~/newproject
ghid init myalias
# → git init
# → user.name = myalias      (local-only)
# → user.email = <looked up> (local-only — ~/.gitconfig untouched)
# → credential.helper disabled for this repo (HTTPS leak prevention)
# → pre-locked: pre-push hook refuses any non-github-myalias push
```

The email is looked up from `~/.config/ghidbar/identities.conf` — one
line per identity:

```
# ~/.config/ghidbar/identities.conf
myalias=me+myalias@example.com
otheralias=me+other@example.com
```

…or pass `--email <addr>` explicitly:

```bash
ghid init myalias --email me+myalias@example.com
```

## Key rotation

```bash
ghid rotate myalias
# → Archives old key to ~/.ssh/.ghid-archive/, generates fresh one,
#   prints both new pubkey and old fingerprint so you know what to
#   add and what to delete on github.com/settings/keys.
```

## Machine-wide new-repo guard

`ghid init` / `ghid lock` are per-repo and opt-in — you have to remember to
run them. The guard is the backstop for when you forget:

```bash
ghid guard install     # set a global git init.templateDir pre-commit hook
ghid guard status      # show whether the guard is active
ghid guard uninstall   # remove it
```

Once installed, git copies the hook into **every** new `git init` / `git clone`
(in any terminal — it's enforced by git itself, not shell state). The hook
refuses to commit until the repo has an explicit **local** `user.email`, so a
brand-new repo can never silently inherit your global identity:

```
✗ ghid-guard: this repo has no explicit (local) git identity.
  A commit here would inherit your GLOBAL identity:  you@work.example
  Pick an identity first (prevents opsec leaks):
    ghid init <identity>
    git config --local user.name/.email ...
```

Properties:
- **Existing repos are untouched** — the template only seeds newly-created repos.
- **Non-destructive** — `ghid guard uninstall` removes it; if you already had an
  `init.templateDir`, the guard merges into it rather than overwriting.
- **Deliberate bypass:** `GHID_GUARD_DISABLE=1 git commit ...`.

## Identity isolation guarantee

When `ghid lock <id>` is set on a repo, the unified pre-push hook
(installed by `install-hooks.sh` in the repo root) refuses any push
whose remote URL doesn't go through `git@github-<id>:...`.

The lock survives across:
- Manual `git push` from any terminal
- IDE-integrated push buttons (most IDEs honor git hooks)
- `git push --force` (still subject to pre-push)

The lock does **not** prevent:
- Renaming the remote URL (which `ghid switch` does intentionally) —
  the lock checks identity, not URL stability
- Removing the hook by hand or with `--no-verify` on push (git's
  documented escape hatch)
