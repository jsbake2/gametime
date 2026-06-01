# gametime

Toggle the workstation between focused-gaming mode (`gametime on`) and back-to-work
mode (`gametime off`). Kills distracting / resource-heavy apps, warns about anything
whitelisted that's still hogging the box, optionally asks a local Qwen2.5 instance
for a second opinion, and remembers what it killed so it can put everything back.

## Install

```sh
git clone https://github.com/jsbake2/gametime.git ~/repos/gametime
ln -s ~/repos/gametime/gametime ~/.local/bin/gametime
```

Requires Python 3.10+ and a Linux box with `/proc`. Nothing else is mandatory:

- `nvidia-smi` if you have an NVIDIA card (AMD is read from sysfs)
- `ssh` reachable to the Qwen host if you want AI suggestions

## Use

```sh
gametime on                                    # interactive
gametime on --yes --launcher steam --no-ai     # silent: nuke everything in the list, keep steam, skip AI
gametime on --dry-run                          # show what would happen, change nothing

gametime off                                   # ask A / B / C
gametime off -A                                # A: relaunch what 'on' killed
gametime off -B                                # B: launch the default off-list
gametime off -C                                # C: pick from a combined menu

gametime status                                # show config + last session
```

## What `on` does

1. Samples per-PID CPU for ~0.5 s.
2. Asks which launcher you're using (Steam / Battle.net / neither) and adds the
   *other* launcher's processes to this run's killlist.
3. Warns if anything **whitelisted** (Chrome, YouTube Music, Google Chat, …) is
   above your configured CPU/MEM thresholds — those won't be killed, but you
   probably want to restart them yourself.
4. Kills everything in the persistent killlist (after a confirm, unless `--yes`).
5. Lists the top resource users that are on neither list and asks which to
   terminate. Anything you pick can be added to the persistent killlist so it
   gets killed automatically next time.
6. Sends a process snapshot to **Qwen2.5:14b** running on `10.0.0.16` (via
   `ssh + curl` to the Ollama container) and shows its kill recommendations.
   Skipped silently if the host is unreachable, or with `--no-ai`.
7. Reports swap usage, hung / zombie processes (grouped by parent so leaks are
   actionable instead of noise), GPU utilization, and the busiest NIC.
8. Records what it killed to `last_session.json` so `off` can put it back.

## What `off` does

Restarts apps. You get three options each run:

- **A — previous session.** Whatever was running before the last `gametime on`.
  For `teams-for-linux` instances we read the `--user-data-dir` flag from the
  saved cmdline and pick the right wrapper (`teams-dpg` / `teams-icr`).
- **B — defaults.** The apps listed in `default_apps.json` (Teams DPG, Teams
  ICR, Google Chat, YouTube Music, VS Code by default).
- **C — pick.** A combined menu of (A) and (B), de-duplicated. Enter a
  selection like `1,3,5-7` or `all` to launch only the ones you want.

## Config

Everything lives in `~/.config/gametime/` as plain JSON, created on first run:

| File | Purpose |
|---|---|
| `killlist.json` | Names that get killed by `on`. Grows automatically when you say "yes, remember this." |
| `whitelist.json` | Names that are NEVER killed and trigger high-usage warnings. |
| `default_apps.json` | What `off --default` launches. |
| `last_session.json` | What `on` killed last time. Used by `off --previous`. |
| `config.json` | Thresholds, launcher patterns, AI settings. |

Matching is case-insensitive and checks `comm`, the basename of `exe`, and
`argv[0]`. A name also matches if a process starts with `<name>-` or `<name>.`
(so `steam` catches `steam-runtime-launcher-service` and wine's `steam.exe`).

### Cmdline patterns

For things that don't have a stable binary name, both lists also accept
`cmdline_patterns` — Python regexes matched against the full command line.
Useful for e.g. Chrome PWAs invoked as `chrome --app=…`.

## Qwen integration

The script shells out to:

```
ssh jbaker@10.0.0.16 curl http://172.23.0.2:11434/api/generate -d @-
```

…where `172.23.0.2:11434` is the Ollama container's docker-bridge address.
Change `ai.ssh_host`, `ai.ollama_url`, or `ai.model` in `config.json` to point
somewhere else. Set `ai.enabled` to `false` (or pass `--no-ai`) to skip.
