```
 █████╗  ██████╗██╗    ██╗ █████╗ ██╗  ██╗███████╗
██╔══██╗██╔════╝██║    ██║██╔══██╗██║ ██╔╝██╔════╝
███████║██║     ██║ █╗ ██║███████║█████╔╝ █████╗  
██╔══██║██║     ██║███╗██║██╔══██║██╔═██╗ ██╔══╝  
██║  ██║╚██████╗╚███╔███╔╝██║  ██║██║  ██╗███████╗
╚═╝  ╚═╝ ╚═════╝ ╚══╝╚══╝ ╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝
```

> **stay awake on AC, sleep on battery — macOS**

Tiny launchd daemon that runs `caffeinate -dimsu` while you're on AC, and stops it the moment you unplug. Apple-Silicon-safe — overrides clamshell sleep, so a closed-lid laptop on a charger stays reachable over SSH, VNC, and Tailscale.

Event-driven via `pmset -g pslog` (IOKit power-source notifications). No polling, no idle CPU, no battery drain between transitions.

---

## How it works

```
              pmset -g pslog
            (IOKit power events)
                     │
                     ▼
          ┌──────────────────┐
          │      acwake      │
          │ (launchd daemon) │
          └───┬──────────┬───┘
              │          │
           on AC      on battery
              │          │
              ▼          ▼
   caffeinate -dimsu   kill caffeinate
   (no sleep, ever)    (sleep normally)
```

When on AC, a single `caffeinate -dimsu` child process holds assertions that prevent every sleep type — including the clamshell-close case. When you unplug, the child dies and macOS's default battery-sleep behavior resumes.

## Install

```sh
brew tap pkhr/tap
brew install acwake
sudo brew services start acwake
```

The daemon needs root because it must run as a system-wide `LaunchDaemon` to override clamshell sleep. `brew services` will create `/Library/LaunchDaemons/homebrew.mxcl.acwake.plist` and start it.

Verify:

```sh
brew services list                       # acwake should be 'started'
acwake status                            # shows current power source
sudo tail -f /Library/Logs/acwake.log    # plug/unplug to see reactions
```

Sample log output:

```
2026-05-06 16:55:14  AC      caffeinate pid=42139
2026-05-06 17:30:02  battery caffeinate pid=42139 stopped
```

## Uninstall

```sh
sudo brew services stop acwake
brew uninstall acwake
brew untap pkhr/tap
```

Removes the daemon, the launchd plist, the binary, and the tap. The Mac returns to default `pmset` behavior. The log file at `/Library/Logs/acwake.log` is left in place; remove it manually if you want to.

## What gets installed

| Path | Purpose |
| --- | --- |
| `$(brew --prefix)/bin/acwake` | the script (managed by brew) |
| `/Library/LaunchDaemons/homebrew.mxcl.acwake.plist` | launchd unit (managed by `brew services`) |
| `/Library/Logs/acwake.log` | stdout / stderr |

## Why not `pmset -c disablesleep 1`?

`disablesleep` is system-wide on macOS — there's no real per-power-source scoping despite pmset accepting `-c`/`-b` flags. Setting it globally drains the battery when you unplug with the lid open. acwake gives clean per-power-source behavior with a four-line state machine.

## License

MIT.
