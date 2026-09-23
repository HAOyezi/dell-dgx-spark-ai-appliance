# Run a GB10 (DGX Spark) AI Appliance from a Windows Desktop — No Linux Terminal Required

> A proven two-machine setup: a **Windows desktop (e.g. a Dell) as the human console**, and a
> **NVIDIA DGX Spark (GB10, ARM64) as the compute appliance**. An AI agent (Hermes Agent) runs on
> the Windows side and routes work to the right machine automatically. You talk to the agent in
> chat (Feishu/Slack/CLI) and **never touch a Linux terminal**.
>
> All paths, IPs and account names in this repo are placeholders. No personal information is included.

## The problem this solves

The GB10 (DGX Spark) is a powerful 121 GB unified-memory ARM64 machine — great for ML training,
large-scale data processing, model serving, rendering, and even games. But it is a **headless
Linux box**: no display, no GUI to speak of, and daily use normally means SSH terminals,
`scp`, `sudo`, journal logs, and friends. That friction is exactly where most non-Linux users
give up.

This pattern removes the friction entirely:

- The **Windows desktop** is where a human works: browser, office apps, chat, and the AI agent.
- The **GB10** is an appliance: it just runs things — training jobs, LLM servers, builds, game hosts.
- The **AI agent on Windows** decides per task which machine to use, executes remotely over SSH,
  and reports results back in plain language. The human only states intent in chat.

```
  Human
   |  (chat: Feishu / CLI / anything)
   v
+------------------------------------------------------+
|  WINDOWS DESKTOP (e.g. Dell)  —  "the console"       |
|  - Hermes Agent (the brain / router)                 |
|  - Browser, office, Moonlight game client            |
|  - Light tasks run here: quick scripts, file edits   |
+------------------------------------------------------+
   |  SSH (key auth) + scp / SFTP
   |  agent routes heavy work down automatically
   v
+------------------------------------------------------+
|  GB10 / DGX Spark (ARM64)  —  "the appliance"        |
|  - 20 cores, 121 GB unified memory, Blackwell GPU    |
|  - RL / ML training, dataset processing              |
|  - LLM serving (sglang / vLLM / Ollama)              |
|  - builds, rendering, batch jobs                     |
|  - optional: Steam game host + Sunshine streaming    |
+------------------------------------------------------+
```

## Division of labor

| Workload | Where | Why |
|---|---|---|
| Quick lookups, file edits, small scripts, drafting | Windows | fast, zero overhead, no context switch |
| Chat with the agent, approvals, reviewing results | Windows | it's where the human is |
| RL / ML training (CPU or GPU) | GB10 | 20 cores + 121 GB RAM; GPU available if not occupied |
| LLM inference serving | GB10 | big unified memory; dedicated serving engine |
| Big data / dataset ETL, compiles, rendering | GB10 | RAM + throughput |
| Long-running background jobs | GB10 | survives desktop reboots; agent polls via SSH |
| Games (Steam) + streaming to desktop | GB10 host, Windows client | see companion repo below |

## One-time setup

### 1. On the GB10 (the appliance)

```bash
# SSH with key auth (do this once, from the Windows side, see step 2)
# Install whatever the appliance must run, e.g.:
#   ML:     python3, torch, stable-baselines3, ...
#   LLM:    sglang / vLLM / Ollama
#   Games:  sudo snap install steam --stable   (optional, see companion repo)
#   Stream: Sunshine (optional)
```

Keep the appliance headless. A display manager (GDM) is only needed for the game/streaming
use case.

### 2. On the Windows desktop (the console)

1. Install **Hermes Agent** (or any agent with terminal + file tools) and connect your chat
   channel (Feishu, Slack, CLI, ...).
2. Set up SSH key auth to the appliance:

   ```powershell
   # generate a key if you don't have one
   ssh-keygen -t ed25519
   # copy the public key to the GB10 (one password entry, ever)
   type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh gb10_user@GB10_IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
   ```

3. Tell the agent the routing rules (a short memory/instruction note is enough):

   > "Light tasks run locally on this Windows machine. Heavy tasks (training, serving,
   > big data, builds, games) run on the GB10 over SSH at GB10_USER@GB10_IP.
   > Never show me raw terminal output unless I ask — summarize results."

That's the entire setup. No VMs, no WSL, no RDP.

## Daily use (what the human actually does)

You type intentions in chat; the agent routes and executes:

| You say in chat | What the agent does |
|---|---|
| "Check disk usage on the Spark" | `ssh` → `df -h` → reports a one-line summary |
| "Start the RL training, 4000 steps" | copies the local project down (scp), launches in a background process on GB10, polls it, reports progress/curve |
| "Serve the 27B model on port 8888" | starts the serving engine on GB10, health-checks, tells you the endpoint |
| "Process this 80 GB dataset" | runs on GB10 (RAM), streams progress, returns results |
| "Launch Kerbal on the Spark" | starts the game on GB10, you watch it on the desktop via Moonlight |

The agent handles the unglamorous parts — SSH session state, background job survival, log
tailing, retrying on transient network blips — which is precisely the part that kills
"regular person uses Linux" attempts.

## Routing heuristics that work

1. **CPU-seconds heuristic**: > ~5 minutes of CPU time → remote.
2. **Memory**: needs > desktop RAM (or any part of the 121 GB pool) → remote.
3. **GPU**: anything GPU-bound → remote (Windows side rarely has a datacenter GPU).
4. **Durability**: must survive a laptop/desktop reboot → remote.
5. **Network egress to overseas CDNs**: if the appliance has a tunnel/proxy configured,
   heavy downloads go there; otherwise download on the desktop and `scp` up.
6. Everything else → local. Latency and simplicity win.

## Games & streaming (optional add-on)

Make the GB10 a game host and play from the Windows desktop:

```
GB10 GPU → Sunshine (NVENC) → LAN → Moonlight (on the Windows desktop)
```

Full measured guide (install, pitfalls, verified game list):
**[gb10-steam-gaming](https://github.com/HAOyezi/gb10-steam-gaming)** — includes a ready-to-install
AI skill package so the agent on the desktop can set the whole thing up for you.

## Why this beats the alternatives

| Alternative | Problem |
|---|---|
| Direct Linux daily use | keyboard/UX mismatch; everything is a CLI; steep for most users |
| WSL / RDP / VNC | extra layers, clipboard/latency pain, still "you drive the terminal" |
| Everything on one machine | you pick the weakest of the two (GPU or convenience) |
| **This: console + appliance + agent routing** | human stays in a familiar Windows/chat world; appliance does muscle work; agent is the driver |

## Requirements & notes

- Windows 10/11 desktop with a working SSH client (built into Win10+).
- GB10 / DGX Spark with Ubuntu 24.04 ARM64 (stock) or similar.
- LAN (or a stable tunnel) between the two machines; key-based SSH.
- Hermes Agent (or compatible) on the Windows side.
- This document contains **no real IPs, usernames, passwords or ports**. Replace the
  placeholders (`GB10_IP`, `GB10_USER`, `SUDO_PASS`, `PROXY_PORT`, `STREAM_USER`/`STREAM_PASS`)
  with your own values.
