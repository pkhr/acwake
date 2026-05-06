# acwake

```
   ╔══════════════════════════════════════════════════════╗
   ║                       acwake                         ║
   ║       stay awake on AC, sleep on battery (macOS)     ║
   ╚══════════════════════════════════════════════════════╝
```

Tiny launchd daemon that runs `caffeinate -dimsu` while you're on AC, and stops it the moment you unplug. Apple-Silicon-safe — overrides clamshell sleep, so a closed-lid laptop on a charger stays reachable over SSH / VNC / Tailscale.

Event-driven via `pmset -g pslog` (IOKit power-source notifications). No polling, no idle CPU, no battery drain between transitions.

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
git clone https://github.com/pkhr/acwake.git
sudo ./acwake/acwake install
```

## Usage

```
acwake install     install and start the launchd daemon
acwake uninstall   stop daemon, remove plist, binary, log
acwake status      show install/runtime state and current power source
acwake run         foreground mode (used internally by launchd)
```

Watch it react in real time:

```sh
sudo tail -f /Library/Logs/acwake.log
```

```
2026-05-06 16:55:14  AC      caffeinate pid=42139
2026-05-06 17:30:02  battery caffeinate pid=42139 stopped
```

## Uninstall

```sh
sudo /usr/local/bin/acwake uninstall
```

Removes the binary, the launchd plist, and the log file. Your Mac returns to default `pmset` behavior.

## What gets installed

| Path | Purpose |
| --- | --- |
| `/usr/local/bin/acwake` | the script (copy of this file) |
| `/Library/LaunchDaemons/acwake.plist` | launchd unit (`RunAtLoad` + `KeepAlive`) |
| `/Library/Logs/acwake.log` | stdout/stderr |

## Why not `pmset -c disablesleep 1`?

`disablesleep` is system-wide on macOS — there is no real per-power-source scoping despite pmset accepting `-c`/`-b` flags for it. Setting it globally drains the battery when you're unplugged with the lid open. acwake gives you clean per-power-source behavior with a four-line state machine.

## License

MIT.
