# Running Hermes Agent in a Lima VM with Telegram, Reachable Through Lid-Close

A runbook for the actual setup in use: Hermes Agent running inside a Lima
VM (not directly on the Mac), talking to Telegram, reachable from a phone
even when the Mac's lid is closed. Written so this can be rebuilt from
scratch without re-deriving each decision, and so the open/unresolved
parts aren't lost.

---

## Why a VM instead of running Hermes directly on the Mac

Running Hermes inside a Lima (Linux) VM instead of natively on macOS
means:
- The gateway is a systemd-supervised Linux service — auto-restart on
  crash, survives logout, standard Linux service semantics instead of
  launchd quirks.
- The Mac's screen lock/sleep behavior no longer directly kills the
  Hermes process, since it's not running on the Mac's own OS session.
- Trade-off: the VM still runs *on* the Mac's hardware, sharing its
  network interface and (for a VM, generally) suspending when the whole
  Mac sleeps. Lid-close protection still has to be solved at the Mac
  level — moving Hermes into a VM does not by itself fix that. Sections
  below cover both halves.

---

## 1. Provision the Lima VM

Check real free disk space before creating anything — on macOS, `df -h /`
can be misleading because multiple APFS volumes (System, Data, VM,
Preboot, Update) share one physical container:

```bash
df -h /
diskutil apfs list | grep -E "Capacity|Free Space"   # the real ceiling, trust this over df
du -sh ~/.lima/*/                                     # per-instance usage
limactl list                                          # STATUS column: Running vs Stopped
```

Create the VM (raw `limactl`, or a project `vm` wrapper if one exists —
prefer the wrapper if present, it can add credential injection and
golden-VM cloning):

```bash
limactl create --name=hermes-vm template://ubuntu-lts
limactl start hermes-vm
```

**Timeout pitfall:** first-time provisioning (image download, disk
expand, boot) can take several minutes. Run with a generous timeout
(400s+) or in the background — a short default timeout will kill it
mid-provision.

Verify outbound networking before installing anything:

```bash
limactl shell hermes-vm -- bash -lc \
  'curl -s -o /dev/null -w "%{http_code}\n" https://api.telegram.org'
```

---

## 2. Install Hermes inside the VM

1. Ubuntu's default `python3` is usually too new for Hermes's pinned
   version — don't fight this with `apt`. Install `uv` first (flag this
   as a pipe-to-shell action needing approval):
   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```
2. Run the official installer (handles Python version, venv, launcher):
   ```bash
   curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
   ```
   or, if working from a git checkout:
   ```bash
   git clone https://github.com/NousResearch/hermes-agent.git ~/.hermes/hermes-agent
   cd ~/.hermes/hermes-agent
   printf "\n" | ./setup-hermes.sh   # skip the interactive wizard if not configuring keys yet
   ```
3. Run the setup wizard / pick a model provider, or configure directly:
   ```bash
   hermes setup
   hermes model
   hermes doctor        # health check
   ```
4. To carry over persona/memory from an existing setup without copying
   secrets, **create the destination directory first** —
   `limactl copy` does not create parent dirs and fails with
   `rsync: change_dir failed: No such file or directory` otherwise:
   ```bash
   limactl shell hermes-vm -- bash -lc "mkdir -p ~/.hermes/memories"
   limactl copy ./SOUL.md hermes-vm:~/.hermes/SOUL.md
   limactl copy ./MEMORY.md hermes-vm:~/.hermes/memories/MEMORY.md
   ```
   Deliberately do **not** copy `config.yaml`, `.env`, or `auth.json`
   unless explicitly intended — those carry live credentials, and moving
   them to a second machine is a separate security decision.

### Known pitfall: a `gh` credential-injection wrapper can break Copilot-provider auth

If the VM was created via a wrapper that auto-injects a sandbox GitHub
App token and shims `gh` to always re-source it, and Hermes is configured
with `provider: copilot` (which shells out to `gh` for its OAuth token),
Hermes silently picks up the sandbox App token instead of a real Copilot
session. Symptom: `HTTP 400: ... GitHub App Server-To-Server Tokens are
not supported for this endpoint`, even though `hermes status` looks
correct. Confirm with `gh auth status` inside the VM — a
`*-sandbox-app[bot]` account means this is the cause, not a Copilot/Nous
auth problem. Fix (either, both reversible):
- Disable/rename the injected `~/.local/bin/gh` wrapper so `gh` resolves
  to the real binary (appropriate for a durable, non-disposable VM).
- Or run `gh auth login` inside the VM for a real Copilot OAuth session.

---

## 3. Configure the Telegram channel

```bash
hermes dashboard        # web admin panel has a messaging-channels setup flow
```
or directly via config — get a bot token from @BotFather on Telegram,
then:

```bash
hermes config set channels.telegram.enabled true
hermes config set channels.telegram.bot_token <token>   # or store in .env — never in config.yaml
```

Secrets belong in `.env` under `$HERMES_HOME`, not in `config.yaml`
directly — never hand-edit `config.yaml` for a token; use `hermes config
set` or the dashboard so a stray indent doesn't corrupt the file and
break the live gateway.

Test the bot responds before moving on to the systemd/lid-close work —
message it from a phone and confirm a reply, while still running Hermes
in the foreground (`hermes gateway start` or a temporary `hermes chat`)
so any auth/token error surfaces immediately rather than being masked by
service-supervision retry behavior.

---

## 4. Run the Hermes gateway as a systemd service inside the VM

Once Telegram responds correctly in a foreground test, install it as a
persistent Linux service — this is the Linux equivalent of launchd
keep-alive:

```bash
hermes gateway install     # creates/enables a systemd --user service, starts it
hermes gateway status      # confirm: Active: active (running) + "Systemd linger is enabled"
```

"Linger enabled" means the service survives the VM user logging out and
restarts across VM reboot. It's fine to install this even before every
channel is fully configured — it idles harmlessly with nothing to listen
for.

If Hermes needs to be reachable via a web dashboard/desktop from the Mac
side too, note that Lima only auto-forwards guest loopback ports to the
host while the discovering `limactl shell` session stays alive — start
services detached and open a separate durable tunnel if needed:

```bash
limactl shell hermes-vm -- bash -lc \
  'nohup hermes dashboard --no-open > /tmp/dashboard.log 2>&1 & disown'
ssh -F ~/.lima/hermes-vm/ssh.config -N -L <port>:127.0.0.1:<port> lima-hermes-vm
```

---

## 5. The actual lid-closed problem: what fails and what doesn't

**Confirmed observed behavior:** when the Mac's lid is closed, the VM
itself does not freeze — the gateway process inside it keeps running.
What breaks is the network path: the Mac's Wi-Fi interface drops
Telegram's long-poll TCP connection for roughly 2-4 minutes after lock,
even though ICMP ping to the Mac can still succeed during that window
(ping succeeding is misleading — it does not mean the long-poll
connection survived).

This means the fix has to target **macOS's networking-under-lock/sleep
behavior**, not the VM or the Hermes process — a systemd-supervised
gateway that never crashes still can't help if the underlying network
socket got torn down by the OS.

### What's been tried

- `sudo pmset -a networkoversleep 1` — helped, but did **not** fully
  resolve the delay. Still under investigation.
- Currently testing: the macOS Battery setting "Prevent Automatic Sleeping
  When Display Is Off" (System Settings → Battery/Energy, or
  `pmset -a disablesleep=1` as the blunter systemwide equivalent) as an
  alternative/additional lever.

### Two separable concerns, don't conflate them

1. **Does the Mac go to full system sleep at all?** Screen lock alone
   does not stop background processes on macOS — only true system sleep
   (idle timeout, lid close beyond a grace period) does. If full sleep is
   confirmed to be happening, prevent it:
   ```bash
   pmset -g | grep sleep              # check current assertions
   ```
   Prefer a reversible, managed approach (a launchd `caffeinate -s` job)
   over a blunt permanent `pmset disablesleep=1`, unless the user
   explicitly wants a permanent systemwide change regardless of
   reversibility:
   ```xml
   <!-- ~/Library/LaunchAgents/com.hermes.caffeinate.plist -->
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
       <key>Label</key><string>com.hermes.caffeinate</string>
       <key>ProgramArguments</key>
       <array><string>/usr/bin/caffeinate</string><string>-s</string></array>
       <key>RunAtLoad</key><true/>
       <key>KeepAlive</key><true/>
       <key>StandardOutPath</key><string>/tmp/hermes-caffeinate.log</string>
       <key>StandardErrorPath</key><string>/tmp/hermes-caffeinate.log</string>
   </dict>
   </plist>
   ```
   ```bash
   launchctl unload ~/Library/LaunchAgents/com.hermes.caffeinate.plist 2>/dev/null
   launchctl load ~/Library/LaunchAgents/com.hermes.caffeinate.plist
   pgrep -fl caffeinate                 # confirm running
   pmset -g | grep sleep                # confirm "sleep prevented by ... caffeinate"
   ```
   `caffeinate -s` blocks system sleep only, not display sleep — the
   screen still locks/blanks on schedule, which is generally desired.
   Rollback:
   ```bash
   launchctl unload ~/Library/LaunchAgents/com.hermes.caffeinate.plist
   rm ~/Library/LaunchAgents/com.hermes.caffeinate.plist
   ```

2. **Even if full sleep is prevented, does Wi-Fi itself get suspended or
   throttled on lock, independent of system sleep state?** This is the
   part actually observed in this setup (the 2-4 min delay) and is not
   fully solved by preventing system sleep alone — `networkoversleep=1`
   is aimed at this but only partially helped. This is the open thread:
   test whether "Prevent Automatic Sleeping When Display Is Off" changes
   the delay, and whether it differs from `disablesleep=1`'s effect.

### Verification checklist for a lid-close test

Don't declare this fixed on one clean run — the symptom has previously
resisted a single passing test overriding repeated real-world reports.
For each candidate fix:
- Note current `pmset -g` assertions and the exact setting changed.
- Close the lid, wait a fixed interval (e.g. 5 min), send a Telegram
  message from a phone, and **time the actual reply latency**, not just
  whether a reply eventually arrives.
- Repeat across multiple lock durations (30s, 2 min, 10 min) — the
  original delay was specifically in the 2-4 min range, so a test that
  only waits 30s could falsely look like a fix.
- Confirm `limactl shell hermes-vm -- bash -lc 'systemctl --user status
  <hermes-gateway-unit>'` shows the gateway was never dead during the
  test — if latency improves but the process had actually restarted, that
  points to a different mechanism (crash-recovery masking, not the
  network delay actually resolving).

---

## Pitfalls recap (quick reference)

- `df -h /` can lie on macOS with shared APFS volumes — use `diskutil
  apfs list` for the real ceiling.
- First-time VM provisioning needs a long timeout or background run.
- `limactl copy` requires the destination directory to already exist.
- A `gh` wrapper injecting a sandbox GitHub App token can silently break
  Copilot-provider OAuth inside the VM — check `gh auth status` if auth
  errors look inconsistent with `hermes status`.
- Screen lock alone does not stop background processes on macOS; only
  true system sleep does — these are separate problems with separate
  fixes.
- Ping succeeding during lock does **not** prove the Telegram long-poll
  connection survived — verify actual message round-trip latency, not
  just host reachability.
- `networkoversleep=1` helped but did not fully fix the delay on its own
  — treat as partial, not a confirmed solution.

## Open items (state honestly — not yet resolved as of this writing)

- Root cause of the 2-4 min Wi-Fi/long-poll delay after Mac lock is not
  fully diagnosed. `networkoversleep=1` is a partial mitigation.
- Whether "Prevent Automatic Sleeping When Display Is Off" (Battery
  settings) meaningfully changes the delay compared to
  `pmset disablesleep=1` has not been confirmed — this is the next test.
- No confirmed final fix as of this writing — this section should be
  updated once a lid-close test at multiple durations comes back clean.
