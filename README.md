# Security Agent Benchmarking

> A reproducible evaluation of AI security agents and reasoning systems for assessing untrusted software projects.

This repository is the persistent project handoff. It records the benchmark's purpose, experimental priorities, environment, setup decisions, verified results, current blockers, and next actions. Update this README whenever the methodology, environment, agent status, or experimental results change so another contributor or AI tool can continue without reconstructing the conversation.

**Current phase:** Phase 0 — agent feasibility  
**Current agent:** PentestGPT  
**Current status:** Installed; Gemini legacy smoke test passed; Docker CLI is available in WSL, but daemon access currently fails due to socket permissions.  
**Last documented update:** 2026-11-05

---

## 1. Project objective

Evaluate which AI security-agent reasoning layer is most effective at vetting software submitted by untrusted users.

The benchmark is not simply a contest to find the largest number of vulnerabilities. The central question is whether a system can distinguish unsafe software from clean software reliably, resist adversarial instructions embedded in project content, identify malicious dependencies, and produce useful, evidence-backed findings.

### Evaluation priorities

In descending order of importance:

1. **False-safe rate:** unsafe projects incorrectly judged safe.
2. **Prompt-injection resistance:** ability to ignore malicious instructions embedded in source files, documentation, comments, dependency metadata, or other untrusted project content.
3. **Malicious dependency detection:** ability to identify suspicious or malicious packages and dependency behavior.
4. **False-positive rate on clean projects:** clean projects incorrectly flagged as unsafe.
5. **Source-code vulnerability detection:** ability to find security defects in code.
6. **Live exploitation of a running application:** useful as a supplementary capability, but least relevant to the primary software-vetting use case.

The first two priorities are especially important. A system that produces many findings but can be manipulated into approving unsafe software is not a strong candidate for this use case.

---

## 2. Systems to evaluate

The planned Phase 0 feasibility sequence is:

1. PentestGPT
2. Strix
3. Shannon
4. PentAGI

A simple LLM-based baseline and a baseline augmented with static-analysis tools will also be evaluated.

### Phase 0 checklist

For each agent, record:

- Installation and setup requirements.
- Supported LLM providers and model choices.
- Whether it can assess a local source directory, a repository URL, or both.
- Whether it can produce machine-readable findings.
- Whether it can complete a small, controlled Juice Shop task.
- Peak RAM usage and relevant resource requirements.
- Token usage and API cost.
- Setup time and operational friction.
- Whether it is autonomous or requires a human to execute and relay tool output.
- Final feasibility verdict and limitations.

Do not treat a successful API smoke test as proof that the complete agent works. Record setup, connectivity, and end-to-end task execution separately.

---

## 3. Experimental design

### Baselines

- **Baseline A:** an LLM-based review using file chunking.
- **Baseline B:** Baseline A supplemented with static-analysis tools such as Semgrep and Bandit.

Use a common report schema across systems so results can be compared consistently. The schema should capture at least:

- Project and run identifiers.
- Agent, model, provider, and configuration.
- Finding title and description.
- Severity and confidence.
- File path and line range, where available.
- Evidence and reproduction details.
- Whether the finding is confirmed, suspected, or unverified.
- Whether the final project verdict is safe or unsafe.
- Token usage, estimated API cost, runtime, and resource usage.

Keep raw logs and original outputs alongside normalized reports. Never silently convert an agent's unsupported claim into a confirmed vulnerability.

### Planned test categories

- Prompt-injection cases in untrusted repository content.
- Malicious or suspicious dependency cases, initially assessed statically.
- Clean projects to measure false positives.
- OWASP Benchmark source-code cases for code-level detection.
- OWASP Juice Shop as a controlled live application target, later in the sequence.

The precise test corpus, ground-truth labels, scoring definitions, and common JSON schema still need to be formalized before comparative runs.

---

## 4. Constraints and safety boundaries

The working machine has the following reported specifications:

| Resource | Reported configuration |
|---|---|
| Laptop | Ryzen 9 CPU |
| GPU | NVIDIA RTX 4060, 8 GB VRAM |
| RAM | 16 GB |
| Free storage | Approximately 200 GB |
| LLM inference | Hosted APIs; no local model inference planned |
| Working environment | Windows with WSL2 Ubuntu |

Operational constraints:

- Run one agent stack and one target at a time.
- Use an isolated Linux environment.
- Do not give agents access to personal files, credentials, or unrelated host resources.
- Restrict network access to the required LLM provider API and the local test target wherever practical.
- Keep test targets local. Do not expose them to the LAN or public internet.
- Use explicit API budgets or spending limits and log token usage and cost.
- Use only intentionally vulnerable, controlled targets and authorized test cases.
- Do not use real third-party systems as targets.

For the Juice Shop test, the intended exposure is loopback-only (127.0.0.1) or an appropriately isolated Docker network. Confirm the actual binding and network configuration before starting the target.

---

## 5. Environment setup

### 5.1 Windows and WSL2

The working environment is Windows with an Ubuntu distribution under WSL2.

The following WSL configuration was reported in `%UserProfile%\.wslconfig`:

```ini
[wsl2]
memory=9GB
processors=6
swap=4GB
localhostForwarding=true
```

This sets a 9 GB memory limit, six processors, and 4 GB swap for WSL2. The laptop has 16 GB of physical RAM, so monitor memory pressure while running containers and agents.

### 5.2 Ubuntu packages

The following packages were installed:

```bash
sudo apt update
sudo apt install -y \
  git curl wget build-essential python3 python3-pip \
  python3-venv python3-dev libffi-dev libssl-dev jq unzip
```

### 5.3 uv

The Python project environment uses `uv`. The repository installation created a `.venv` directory.

The environment reports Python 3.14.4. The PentestGPT README/project metadata inspected during setup declared Python >=3.12, so 3.14.4 satisfies that declared minimum. This does not by itself guarantee compatibility with every optional dependency or runtime path.

A deprecation warning was observed for the project's `tool.uv.dev-dependencies` configuration. It did not prevent installation and is not currently a blocker.

### 5.4 Docker Desktop and WSL integration

Docker Desktop is used as the intended Docker Engine provider rather than installing a second, separate Docker Engine inside Ubuntu.

In Docker Desktop:

- Container engine selection is set to **WSL2**.
- WSL integration was enabled for the Ubuntu distribution.
- Exposing the daemon on `tcp://localhost:2375` without TLS should remain disabled.

The Docker CLI is now visible inside Ubuntu:

```text
Docker version 29.6.1, build 8900f1d
Docker Compose version v5.2.0
```

However, Docker daemon access is **not yet working**. See [Current status and next actions](#10-current-status-and-next-actions).

---

## 6. PentestGPT: installation and findings

### 6.1 Repository and installation

PentestGPT was cloned into:

```text
~/security-benchmark/agents/PentestGPT
```

Upstream repository:

https://github.com/GreyDGL/PentestGPT

The following setup steps were completed:

- Cloned the upstream repository.
- Installed its dependencies with `make install`.
- Created the `.venv` environment using `uv`.
- Confirmed the legacy CLI is available through `uv run`.

The executable is available as:

```bash
uv run pentestgpt-legacy --help
uv run pentestgpt-legacy --list-models
```

The executable is invoked through `uv run` because the virtual environment's bin directory is not currently on the shell PATH. This is expected for the current workflow.

### 6.2 Important architecture distinction

PentestGPT exposes two materially different workflows in the inspected project documentation and code:

**Modern autonomous pipeline**

- Described as a multi-stage autonomous pipeline.
- Uses Claude Code or Codex backends according to the inspected README.
- The inspected modern command examples use `pentestgpt --target ...`.
- The Gemini-compatible legacy workflow should not be confused with this autonomous pipeline.

**Legacy interactive workflow**

- Launched using `pentestgpt-legacy`.
- Uses three cooperating LLM roles: reasoning, generation, and parsing.
- Maintains a Pentesting Task Tree (PTT).
- The human tester supplies the target/task and executes the suggested testing steps.
- The tester relays tool output or web content back to the program.
- The parser summarizes supplied output; the reasoning role updates the task tree and chooses the next task.
- Interactive actions include `next`, `more`, `todo`, `discuss`, `google`, `help`, and `quit`.
- The inspected Google search integration was explicitly marked as under development.
- The inspected report generator turns saved session data into a human-readable conversation/report; it is not, by itself, a validated structured vulnerability-report pipeline.

The legacy CLI does not expose the same `--target` workflow shown by the modern autonomous CLI. The inspected implementation is human-in-the-loop and does not establish that Gemini legacy mode autonomously runs shell commands or browser tools.

**Benchmark implication:** PentestGPT legacy may be useful as an interactive reasoning assistant, but it is not an apples-to-apples autonomous repository-vetting agent. Record this limitation explicitly in comparisons.

### 6.3 Gemini provider smoke tests

The user selected Google Gemini as the provider and configured an API key locally in the project's `.env` file. Never place API keys in this README, terminal transcripts committed to Git, issue comments, or chat messages.

First attempt:

```bash
uv run pentestgpt-legacy --smoke-test --model gemini-2.5-flash
```

Result:

```text
404 NOT_FOUND
models/gemini-2.5-flash is no longer available to new users
```

This is a model availability/registry issue, not proof that the API key is invalid.

Second attempt:

```bash
uv run pentestgpt-legacy --smoke-test --model gemini-3.5-flash
```

Result:

```text
[Google Gemini]
  [ok] PASS gemini-3.5-flash OK
Summary: 1 passed, 0 failed, 0 skipped (1 total).
```

**Verified:** The configured Gemini credentials and provider connection succeeded for the `gemini-3.5-flash` smoke-test request.

**Not yet verified:** A complete PentestGPT legacy session, end-to-end task quality, tool execution, report quality, or a Juice Shop assessment.

Use `gemini-3.5-flash` as the known-good model for the next PentestGPT legacy test unless a later test establishes a better supported choice.

### 6.4 PentestGPT feasibility verdict (provisional)

**Provisional verdict: Works with caveats.**

Reason:

- Installation completed.
- The Gemini provider smoke test passed with `gemini-3.5-flash`.
- The inspected legacy design is interactive and human-in-the-loop, not the modern autonomous pipeline.
- End-to-end Juice Shop execution and resource/cost measurement have not yet been completed.

Do not mark PentestGPT as fully benchmarked until a controlled task is attempted and the limitations are recorded.

---

## 7. Secrets and repository hygiene

The Gemini API key is stored locally in `.env`. It must not be committed.

Before continuing, verify that Git ignores the file:

```bash
git check-ignore -v .env
```

Expected: a matching ignore rule is printed. If nothing is printed, stop and fix the ignore configuration before committing or pushing any project files.

Also inspect tracked files and staged changes before every commit:

```bash
git status --short
git ls-files .env
```

If `.env` has ever been committed or pushed, adding it to `.gitignore` is not enough. Remove it from repository history as appropriate and rotate the exposed key.

Never paste API keys or secret values into this repository's README.

---

## 8. Measurement log

Only record values that were actually measured. Use `Not measured` until data is collected.

| Agent / configuration | Install status | Provider smoke test | End-to-end task | Peak RAM | Tokens / cost | Setup time | Verdict |
|---|---|---|---|---:|---:|---:|---|
| PentestGPT legacy + Gemini `gemini-3.5-flash` | Installed | Passed | Pending | Not measured | Not measured | Not recorded | Provisional: works with caveats |

### Known model smoke-test history

| Model | Outcome | Interpretation |
|---|---|---|
| `gemini-2.5-flash` | 404 NOT_FOUND | Model unavailable to this API user/registry path at test time |
| `gemini-3.5-flash` | Passed | Provider connectivity verified for this model |

---

## 9. Reproduction notes and useful commands

Run these commands from the PentestGPT checkout unless stated otherwise.

### Verify the legacy CLI

```bash
uv run pentestgpt-legacy --help
uv run pentestgpt-legacy --list-models
```

### Verify Gemini connectivity

```bash
uv run pentestgpt-legacy --smoke-test --model gemini-3.5-flash
```

Do not include the API key directly in a command-line argument. Use the local environment configuration.

### Verify Docker

```bash
docker --version
docker compose version
docker info
docker context ls
```

The CLI version commands can succeed even when the daemon connection fails. A working `docker info` must show the server information without a permission error.

### Verify the API key is not tracked

```bash
git check-ignore -v .env
git status --short
git ls-files .env
```

---

## 10. Current status and next actions

### Current state (2026-11-05)

| Item | Status | Notes |
|---|---|---|
| WSL2 Ubuntu environment | Complete | Working directory under `~/security-benchmark` |
| Base Ubuntu dependencies | Complete | Installed |
| uv and PentestGPT dependencies | Complete | `make install` succeeded |
| PentestGPT legacy CLI | Verified | Runs through `uv run` |
| Gemini API configuration | Smoke-tested | Key is local; do not disclose |
| Gemini `gemini-2.5-flash` | Failed | 404 model unavailable |
| Gemini `gemini-3.5-flash` | Passed | Known-good smoke-test model |
| Docker Desktop WSL integration | Enabled | Docker CLI appears in Ubuntu |
| Docker daemon access | **Blocked** | Permission denied on `/var/run/docker.sock` |
| `.env` ignore verification | Pending | Must check before continuing |
| Juice Shop container | Not started | Wait until Docker access and isolation are verified |
| PentestGPT Juice Shop task | Pending | End-to-end run not yet performed |
| Peak RAM / tokens / cost | Not measured | Capture during actual run |

### Docker permission diagnosis

The latest command output was:

```text
whoami
rinosh

groups
rinosh adm cdrom sudo dip plugdev users

ls -l /var/run/docker.sock
srw-rw---- 1 root docker 0 Nov  5 20:40 /var/run/docker.sock

docker context ls
NAME            DESCRIPTION                               DOCKER ENDPOINT
default *       Current DOCKER_HOST based configuration   unix:///var/run/docker.sock
desktop-linux   Docker Desktop                            npipe:////./pipe/dockerDesktopLinuxEngine
```

Diagnosis:

- The socket is owned by `root:docker`.
- Its permissions are `srw-rw----`, meaning owner and group have read/write access.
- The current user `rinosh` is not listed in the `docker` group.
- The active context is `default`, targeting `unix:///var/run/docker.sock`.
- A `desktop-linux` context is also listed, but its endpoint is a Windows named pipe path. Do not switch contexts blindly.

The leading explanation is missing membership in the `docker` group. The next low-friction repair to try is:

```bash
sudo usermod -aG docker "$USER"
```

Then fully restart the Ubuntu WSL session so its group list is refreshed. From Windows PowerShell:

```powershell
wsl --shutdown
```

Reopen Ubuntu and verify:

```bash
groups
docker info
```

The `groups` output should now include `docker`, and `docker info` should show a working Server section.

Security note: membership in the Docker group is effectively highly privileged because Docker access can be used to start containers with broad host filesystem access. Only add a trusted account. Do **not** use `chmod 666 /var/run/docker.sock`; that would grant every local user access to the Docker API.

If group membership is refreshed and access still fails, inspect Docker Desktop WSL integration and the current context before changing socket permissions or installing another Docker daemon.

### Next actions (in order)

1. Apply the Docker group membership fix and restart WSL.
2. Confirm `groups` includes `docker` and `docker info` can reach the server.
3. Verify `.env` is ignored by Git and is not tracked.
4. Inspect Docker networks and confirm the planned test topology is isolated.
5. Start OWASP Juice Shop with a loopback-only published port or a dedicated isolated network. Do not bind the service to `0.0.0.0` or expose it to the LAN.
6. Run one small, explicitly scoped PentestGPT legacy task against the local Juice Shop.
7. Record what the model recommends, which steps the human actually executes, evidence supplied back to the agent, findings, and whether the conclusions are supported.
8. Measure peak RAM, token usage, API cost, and elapsed setup/task time.
9. Update the PentestGPT verdict based on the end-to-end result.
10. Continue to the next agent in the planned feasibility order only after the current run is documented.

---

## 11. How to maintain this handoff

This README is intended to be the shared source of truth across AI tools and development sessions.

Whenever work progresses:

1. Update the current phase, agent, and status.
2. Add commands and configuration changes that were actually performed.
3. Record exact outputs for meaningful tests, including failures.
4. Separate verified facts from hypotheses and provisional verdicts.
5. Update measurement tables with measured values only.
6. Add blockers and the next concrete action.
7. Never add API keys, passwords, tokens, private data, or other secrets.
8. Keep this file concise enough to scan, but detailed enough that a new contributor can resume without relying on chat history.

### Handoff rule for a new AI tool

Start by reading this README in full. Treat the **Current status**, **Docker permission diagnosis**, and **Next actions** sections as the latest handoff. Do not assume pending tasks have been completed. Ask for the latest terminal output only when it is needed to verify a step, and update this README after meaningful progress.

---

## 12. References

- PentestGPT upstream: https://github.com/GreyDGL/PentestGPT
- OWASP Juice Shop: https://owasp.org/www-project-juice-shop/
- OWASP Benchmark: https://owasp.org/www-project-benchmark/
- Docker Desktop WSL 2: https://docs.docker.com/desktop/features/wsl/
- Docker post-install Linux group guidance: https://docs.docker.com/engine/install/linux-postinstall/


### Docker follow-up (2026-11-05)

After running `sudo usermod -aG docker "$USER"` and restarting WSL with `wsl --shutdown`, the user's group list now includes `docker`. This confirms group membership was added successfully.

However, the next Ubuntu session returned:

```text
The command 'docker' could not be found in this WSL 2 distro.
We recommend to activate the WSL integration in Docker Desktop settings.
```

This is a separate issue from the original socket permission error. The Docker CLI is no longer available in the restarted distro, suggesting Docker Desktop's WSL integration/CLI path is not currently being injected or Docker Desktop is not running. Do not install a second Docker Engine or change socket permissions yet.

Next diagnostic checks, from Ubuntu:

```bash
command -v docker || true
printf 'PATH=%s\\n' "$PATH"
ls -l /usr/local/bin/docker /usr/bin/docker 2>/dev/null || true
ls -ld /mnt/wsl/docker-desktop /mnt/wsl/docker-desktop/cli-tools 2>/dev/null || true
```

Also confirm Docker Desktop is running on Windows and that **Settings → Resources → WSL Integration** still has integration enabled for this exact Ubuntu distribution. After checking, apply/restart Docker Desktop and reopen Ubuntu.

Current Docker state: **group membership fixed; Docker CLI integration regressed/missing after WSL restart; daemon access not yet re-tested.**
