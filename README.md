# Claude Desktop on Arch / Garuda

Installer for the official Claude desktop app on distributions without `apt`.

## The problem

Claude Desktop for Linux is in beta and is distributed **only** as a `.deb` package from
Anthropic's apt repository. The [official documentation](https://code.claude.com/docs/en/desktop-linux)
requires Ubuntu 22.04+ or Debian 12+, and explicitly states that other distributions are
not supported yet.

On Arch, Garuda, Manjaro, EndeavourOS, Fedora or openSUSE there is no official path. The
usual workarounds are repackaging the Windows installer or trusting a third-party build.

## How it solves it

It does exactly what `apt` would do, by hand: downloads the **official** `.deb` from
Anthropic while verifying the full chain of trust, then installs it with native desktop
integration.

```
GPG key (fingerprint pinned in the script)
  └─> InRelease signed by Anthropic
        └─> Packages hash declared in the signed index
              └─> .deb hash declared in Packages
```

If a single link fails, the script aborts and installs nothing.

### The 12 steps

| Step | What it does |
|:----:|--------------|
| 1 | Checks required tools and maps the kernel architecture (`x86_64`/`aarch64` → `amd64`/`arm64`). Warns if the system does have apt |
| 2 | Downloads the signing key and **compares its fingerprint** against the one pinned in the script |
| 3 | Verifies the signature of the repository's `InRelease` index with `gpgv` |
| 4 | Downloads the package list and checks its hash against the already-verified index |
| 5 | Picks the newest version (`sort -V`) and compares it with the installed one |
| 6 | Downloads the `.deb` and verifies its hash against the package list |
| 7 | Extracts only `data.tar` — the maintainer scripts are never executed |
| 8 | Runs `ldd` on the actual binaries to detect missing libraries, suggesting packages via `pacman -F` |
| 9 | Stages the new version alongside the old one and swaps at the end, so a failure never leaves a half-installed app |
| 10 | Installs the `.desktop` entry with an absolute `Exec`, the icons, refreshes caches and sets up `PATH` |
| 11 | Diagnoses the environment: display server, system keyring and the `claude://` scheme handler |
| 12 | Prints a summary of what was installed |

## Usage

```bash
git clone https://github.com/jason-devcode/claude-desktop-arch-linux.git
cd claude-desktop-arch-linux
./claude-desktop-setup
```

By default it installs into `~/.local`, **without root**. To keep it handy for later runs:

```bash
install -Dm755 claude-desktop-setup ~/.local/bin/claude-desktop-setup
```

It is idempotent: **running it again updates** to the latest version, or does nothing if
you are already up to date. The same command installs and updates.

| Option | Purpose |
|--------|---------|
| *(no options)* | Installs or updates into `~/.local` without root |
| `--system` | Installs into `/opt` for all users (asks for `sudo`, needs a real terminal) |
| `--check` | Only reports whether a new version exists, installs nothing |
| `--force` | Reinstalls even if already up to date |
| `--prefix DIR` | Uses a different prefix instead of `~/.local` |
| `--launch` | Opens the app when finished |
| `--uninstall` | Removes what the script installed |
| `--help` | Help |

Requires `curl`, `gnupg` and `libarchive` (for `bsdtar`). On Arch:
`sudo pacman -S curl gnupg libarchive`

## Notes

**The `.deb` maintainer scripts are deliberately not executed.** They register the apt
repository (useless without apt) and write an AppArmor profile that is only needed on
Ubuntu 24.04+, where the kernel restricts unprivileged user namespaces. Instead, step 9
checks the three switches that can close those namespaces and decides whether
`chrome-sandbox` needs the setuid root bit. Where it isn't needed, it isn't set — one
fewer setuid binary on the system.

**The menu entry's `Exec` is rewritten with an absolute path.** The official `.desktop`
ships `Exec=claude-desktop`, which depends on `PATH`; graphical launchers (rofi, dmenu,
desktop menus) run that value verbatim, and the graphical session does not inherit the
`PATH` of your interactive shell.

**If you use i3, sway, bspwm or similar**, step 11 checks what usually breaks on minimal
desktops: without a Secret Service on the session bus, the app warns that your login will
not be saved and you have to authenticate on every start. The script tells you what to
install and launch if it's missing.

**Not in the Linux beta yet:** Computer Use and voice dictation. The Quick Entry global
shortcut works on X11; on native Wayland it needs your compositor's `GlobalShortcuts`
portal.

Tested on Garuda Linux with i3 (x86_64).
