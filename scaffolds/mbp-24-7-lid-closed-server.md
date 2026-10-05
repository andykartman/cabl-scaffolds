# An old MacBook as a 24/7 AI server, lid closed

*From Drew Dixon, Creating a Balanced Life. This is the actual setup running my AI system, not a tutorial I read.*

My whole AI system runs on a 2018 13-inch MacBook Pro that sits on a shelf with the lid closed. Intel i5, 8 GB of RAM, macOS Sequoia. No external monitor, no HDMI dummy plug, no app keeping it awake. It has been up for four days since its last planned reboot, and the power log shows zero sleep events.

It took me four goes to get it reliable. Not the lid. The lid took one command. The four goes were the things that quietly killed it afterwards, and each one became a watchdog. Here is all of it.

## 1. The lid: one command

macOS wants to sleep a laptop when the lid closes. The clean override is the system-level sleep disable:

```bash
sudo pmset -a disablesleep 1
```

That is the whole trick. With that flag set, the Mac stays awake with the lid closed and nothing else plugged in. The Sleep item in the Apple menu greys out, which is expected. Undo with `disablesleep 0`.

The rest of my power profile, for completeness:

```bash
sudo pmset -a sleep 0 standby 0 hibernatemode 0 powernap 0 disksleep 0
sudo pmset -a displaysleep 10 womp 1 tcpkeepalive 1 ttyskeepawake 1
```

Idle sleep never, standby and hibernate off, display dark after ten minutes, wake-on-network on, keep TCP alive, keep SSH sessions awake.

Check it worked:

```bash
pmset -g | grep SleepDisabled                     # SleepDisabled 1
ioreg -r -k AppleClamshellState | grep Clamshell   # AppleClamshellState = Yes means the lid is closed
pmset -g log | grep -c "Entering Sleep"            # should stay at 0
```

**The honest limit.** This works because nothing I run needs a screen. The jobs are background processes: pollers, small web servers, scheduled scripts, headless AI runs. If you need on-screen things with the lid closed, such as driving a GUI or an iOS simulator, macOS has no display to draw on and a dummy plug or a virtual display is the right fix. I avoid needing a screen rather than faking one.

## 2. How the work runs: launchd, not apps

Everything is a macOS Launch Agent, a plist in `~/Library/LaunchAgents/`. 77 of them on this machine, 63 loaded right now, 17 of them "keep alive" servers that launchd restarts if they die. Two patterns cover nearly everything:

- **Always-on server:** `RunAtLoad` true, `KeepAlive` true. Dies, comes back.
- **Cron-style job:** `RunAtLoad` true, `StartInterval` N seconds.

Stdout and stderr go to `/tmp/<name>.log` and `.err` so I can read what happened after the fact.

Two macOS gotchas that each cost me an afternoon:

- A script that launchd runs cannot live inside `~/Library/CloudStorage` or `~/Library/Mobile Documents`. The system shell is protected and silently fails to read it (exit 126). Keep scripts in `~/.local/bin/` and let them read the synced folders.
- If a job needs Calendar, Contacts or Reminders access, wrap it so a permitted app (Terminal) is the parent, or the permission prompt never fires and the job hangs.

I reach the machine over SSH on the home network and over Tailscale from anywhere. The AI itself runs headless on the box, logged in once with a long-lived token, so sessions do not need my laptop open.

## 3. The four goes: what killed it, and the watchdog each one left behind

**Go one: it went flat.** 13 June. The power cable was knocked out, the battery ran down over hours, and the whole system went dark without a sound. Fix: a power monitor that polls `pmset -g batt` every two minutes and pushes an alert to my phone the moment it switches from mains to battery, again at 20 percent, and once more on recovery.

**Go two: it went flat again, overnight.** 27 August, unplugged at 18:09, found dead in the morning. The two-minute monitor had fired, but once. Fix: a second sentinel that stays resident, checks every four minutes while on battery, and escalates at 25 and 10 percent so I get several chances.

**Go three: the monitoring died with the machine.** Twice, 17 and 27 August. Obvious in hindsight: a watchdog that lives on the server cannot report the server being down. Fix: a watchdog on a *different* machine (my everyday laptop) that probes the server every five minutes over LAN and Tailscale. Three straight misses pages me, repeats at most every six hours, and tells me when it comes back.

**Go four: up for seven days, reachable by nobody.** 2 to 9 September. Uptime fine, load normal, Wi-Fi showed "active", and the box was off the network the entire week. The link was wedged. Toggling Wi-Fi off and on fixed it instantly. Fix: a self-healing watchdog that tests gateway reachability (not SSID, which macOS lies about without Location Services), and after three bad probes power-cycles Wi-Fi, rate-limited to four toggles an hour so a real ISP outage does not make it flap forever.

**The bonus one: silently killed by an upgrade.** 8 September, `brew upgrade python` re-signed the interpreter under a running job. launchd kills a job whose binary changes signature, with empty logs and no exit code you would notice. Fix: a daily scan of `launchctl print` for every job's last exit reason, which reloads anything killed for code-signing.

Every watchdog also writes a heartbeat. A separate daily job lists the jobs whose heartbeat is stale. The principle under all five: **an alert needs a human; a watchdog that heals itself does not.** Alert when a person must act (plug it in). Heal when a script can (toggle Wi-Fi, reload a job).

## 4. The scaffold: give this to your own AI

Paste the block below into your AI's instructions or a project file, then say "set up the always-on Mac".

```
You are helping me run a spare Mac as an always-on server for my AI system.
1. Ask me: lid open or closed, is anything on it screen-dependent, is it on mains, and which second device (phone, laptop) can receive alerts.
2. Power: propose the pmset profile (disablesleep for lid-closed, sleep 0, standby 0, hibernatemode 0, displaysleep short, womp 1, tcpkeepalive 1). Give me the commands and the three checks.
3. Jobs: every background task is a Launch Agent with RunAtLoad, either KeepAlive (server) or StartInterval (cron). Scripts live in ~/.local/bin, never in synced folders. Logs to /tmp/<name>.log.
4. Watchdogs, in this order: mains-power alert (poll pmset -g batt, alert on AC to battery, at 25% and 10%); an off-box probe from my other device (3 misses then alert, repeat every 6 h, recovery notice); a network self-heal that tests the gateway and power-cycles Wi-Fi, rate-limited; a daily launchd exit-reason scan.
5. Every watchdog writes a heartbeat file; one daily job reports stale heartbeats.
6. Rule: alert when a human must act, heal when a script can. Never add a second alert for something a script could fix.
Ship the first three steps today and the watchdogs one at a time as real failures teach me what to watch.
```

That last line is the real lesson. I did not design the watchdogs up front. Each one exists because the machine died in a specific way and I wanted that way to never be silent again.

*Provenance: setup and every failure are mine, Jun–Sep 2026. The scripts were written with Claude. This page was drafted from the live machine and edited by me.*
