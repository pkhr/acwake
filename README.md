```
 █████╗  ██████╗██╗    ██╗ █████╗ ██╗  ██╗███████╗
██╔══██╗██╔════╝██║    ██║██╔══██╗██║ ██╔╝██╔════╝
███████║██║     ██║ █╗ ██║███████║█████╔╝ █████╗  
██╔══██║██║     ██║███╗██║██╔══██║██╔═██╗ ██╔══╝  
██║  ██║╚██████╗╚███╔███╔╝██║  ██║██║  ██╗███████╗
╚═╝  ╚═╝ ╚═════╝ ╚══╝╚══╝ ╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝
```

> **stay awake on AC, sleep on battery — macOS**

A microscopic launchd wrapper around `caffeinate -s`. While you're on AC, system sleep is held off. When you unplug, the assertion goes inert and macOS sleeps normally. **Display sleep, screen saver, dim, and lock all keep working as configured** — only system sleep is prevented.

---

## How it works

```
       caffeinate -s
       (kIOPMAssertPreventSystemSleep, AC-scoped by the kernel)

       on AC      ──►  enforced (no system sleep)
       on battery ──►  inactive (sleep normally)

       display, screen saver, dim, lock:  untouched
```

The kernel itself only enforces a `PreventSystemSleep` assertion while the Mac is on AC power. No event loop, no polling, no AC/battery detection in our own code — `caffeinate(1)` and the IOKit power-management subsystem do all of it.

## Install

```sh
brew tap pkhr/tap
brew install acwake
sudo brew services start acwake
```

The daemon runs as a system-wide `LaunchDaemon` (root). `brew services` writes `/Library/LaunchDaemons/homebrew.mxcl.acwake.plist` and starts it.

Verify:

```sh
brew services list                       # acwake should be 'started'
acwake status                            # power source + assertion state
```

Sample `acwake status` output:

```
Now drawing from 'AC Power'
PreventSystemSleep: active
```

## Uninstall

```sh
sudo brew services stop acwake
brew uninstall acwake
brew untap pkhr/tap
```

Removes the daemon, the launchd plist, and the binary. Mac returns to default `pmset` behavior.

## What gets installed

| Path | Purpose |
| --- | --- |
| `$(brew --prefix)/bin/acwake` | the script (managed by brew) |
| `/Library/LaunchDaemons/homebrew.mxcl.acwake.plist` | launchd unit (managed by `brew services`) |

## Limitations

- **Closed lid on Apple Silicon (Ventura+):** macOS may force sleep when the lid magnet detects closure regardless of any software assertion (including `caffeinate -s` and `pmset disablesleep 1`). If your goal is reachability of a closed-lid laptop, the only Apple-supported path is **clamshell mode with an external display, keyboard, and mouse attached**. Tools like [Amphetamine](https://apps.apple.com/us/app/amphetamine/id937984704) claim to bypass this via a non-public IOKit path; this script does not attempt to.
- **No GUI.** Control surface is `brew services` plus `acwake status`.

## Why not `pmset -c disablesleep 1`?

`disablesleep` is system-wide on macOS — there's no real per-power-source scoping despite pmset accepting `-c`/`-b` flags. Setting it globally drains the battery when you unplug with the lid open. `caffeinate -s` is the IOKit-supported way to express "no system sleep, AC only" without that side effect.

## License

MIT.
