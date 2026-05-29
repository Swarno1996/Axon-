# Axon

**An AI-native system intelligence layer. Self-monitoring, self-healing, human-approved.**

Axon runs alongside your operating system. It watches your system, calls an AI to diagnose anything that looks wrong, proposes fixes with a risk score on each one, applies the safe ones automatically, and holds the risky ones for your approval. Every action is logged.

It is not a new operating system. It is the agent layer that makes any existing system self aware, on Linux, macOS, or Windows.

```
observe  →  analyse  →  decide  →  act  →  verify
  (system)   (rules)     (AI)     (gated)  (re-check)
                                     │
                          low risk ──┴── high risk
                          auto-apply      human approval
```

## Why Axon

Traditional tools stop at the alert. Datadog and New Relic tell you something is wrong. PagerDuty wakes you up. Screaming Frog reports what it crawled. None of them act.

Axon closes the loop. It observes, reasons about what it sees, proposes a concrete fix, and carries it out once a human signs off on anything risky. The intelligence comes from an AI model you choose; Axon is the harness that lets that intelligence touch a real system safely.

## How it works

Each stage of the loop is its own module:

| Stage | Module | What it does |
|-------|--------|--------------|
| Observe | `monitor.py` | Reads CPU, memory, disk, processes, service logs. Read only. |
| Analyse | `context.py` | Cheap threshold rules triage the snapshot. The AI is only called if something is flagged, so cost stays proportional to how much is actually wrong. |
| Decide | `brain.py` | Sends the structured context to the AI, gets back a diagnosis plus actions, each with a risk level. |
| Act | `executor.py` | Low-risk actions run immediately. Everything else goes to the approval queue. |
| Verify | `loop.py` | Re-observes so the next cycle confirms whether the fix held. |

The human-in-the-loop gate (`approval.py` plus the `review` command) is what makes autonomous remediation safe on a real machine.

## Choose your AI brain

Axon supports four backends out of the box. You pick one in `config.yaml` with the `model` setting. The prefix decides the backend.

| Backend | Cost | API key | `model:` value | Install |
|---------|------|---------|----------------|---------|
| Ollama (local) | Free | None | `ollama/llama3.2:1b` | [ollama.com](https://ollama.com) |
| Claude | Paid | `ANTHROPIC_API_KEY` | `claude-opus-4-5` | `pip install anthropic` |
| OpenAI | Paid | `OPENAI_API_KEY` | `gpt-4o-mini` | `pip install openai` |
| Gemini | Paid (free tier) | `GOOGLE_API_KEY` | `gemini-2.0-flash` | `pip install google-genai` |

You only need to install the package for the backend you actually use.

### Free and offline with Ollama

No key, no cost, runs on your own machine. For low-RAM machines (under 8GB free) use a light model:

```bash
# install Ollama from https://ollama.com, then:
ollama pull llama3.2:1b   # ~1.3GB, runs on modest laptops
ollama serve              # leave this running in its own terminal
```

> Heads up: the full `llama3` model needs about 8GB of free RAM and can freeze or restart a smaller machine. Start with `llama3.2:1b`. Step up to `llama3` only if you have the headroom.

## Install

```bash
git clone https://github.com/YOUR_USERNAME/axon.git
cd axon
python -m venv .venv
# Windows PowerShell:  .venv\Scripts\Activate.ps1
# Windows CMD:         .venv\Scripts\activate.bat
# macOS / Linux:       source .venv/bin/activate
pip install -r requirements.txt
```

If PowerShell blocks the activate script, run this once then try again:
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

## Configure

```bash
cp config.example.yaml config.yaml   # Windows: copy config.example.yaml config.yaml
```

Open `config.yaml` and set `model` to the backend you want. Everything else has a working default; `config.example.yaml` documents every option.

If you chose a paid backend, set its key first:

```bash
# macOS / Linux
export ANTHROPIC_API_KEY=your_key_here

# Windows PowerShell
$env:ANTHROPIC_API_KEY = "your_key_here"

# Windows CMD
set ANTHROPIC_API_KEY=your_key_here
```

## Run

Axon has three commands. The one you want most of the time is `once`.

### Run a single check (recommended for manual use)

```bash
python -m axon --config config.yaml once
```

Runs exactly one observe-to-verify cycle and exits. Nothing stays running in the background. Run it whenever you want a check, or wire it into a scheduler.

### Run continuously

```bash
python -m axon --config config.yaml run
```

Repeats a cycle every `interval_seconds` (default 60) until you press Ctrl+C. Use this when you want Axon to be a long-lived daemon.

### Review queued actions

```bash
python -m axon --config config.yaml review
```

Walks you through every medium and high risk action Axon queued, letting you approve or skip each one.

## Schedule it (instead of leaving `run` going)

Many people prefer to let the operating system handle timing and keep Axon as a quick `once` call.

**Linux / macOS (cron, every 5 minutes):**
```cron
*/5 * * * * cd /path/to/axon && .venv/bin/python -m axon --config config.yaml once
```

**Windows (Task Scheduler):** create a Basic Task on your preferred trigger, action "Start a program", program `python`, arguments `-m axon --config config.yaml once`, and set "Start in" to your axon folder.

**systemd timer (Linux):** ship a `axon.service` running `once` plus an `axon.timer` with your schedule. (Template planned for v0.3.)

## Safety model

- The monitor never writes. It only reads.
- The executor is the single place that runs commands, so all safety checks live in one file.
- Low risk means read-only or trivially reversible. Medium and high risk always wait for a human, regardless of config.
- Every observation, decision, and action is written to an append-only audit log in JSON lines format at `audit_path`.

## Tests

```bash
pip install -r requirements-dev.txt
pytest
```

The suite covers the analyse stage, the decision parser, and the brain routing. It runs with no API key, no installed backend, and no network.

## Roadmap

- [x] v0.1: Linux daemon, Claude brain, system metrics, approval CLI, audit log
- [x] v0.2: `once` mode, Ollama / OpenAI / Gemini brains, light-model support
- [ ] v0.3: macOS and Windows native monitors, systemd timer template
- [ ] v0.4: web platform mode, crawl and analyse a live URL
- [ ] v0.5: web dashboard replacing the review CLI
- [ ] v0.6: multi-agent coordination across a fleet of servers
- [ ] v0.7: MCP tool layer for richer remediation

## License

MIT. See [LICENSE](LICENSE).

## Status

Early-stage proof of concept. The core loop runs end to end. Contributions and forks welcome.
