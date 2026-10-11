# Slowpoke — Spec v1

Oct 11, 2026 · @Shubham

## Summary

Slowpoke is a local-first, always-on recorder that explains, after the fact, why a workstation got slow or misbehaved. v0.1 targets Windows and answers two questions: `slowpoke why` for the last auto-detected incident, and "what spawned that window that just flashed?"

The thesis: resource usage is not root cause. Task Manager shows what is consuming resources now. Slowpoke reconstructs what changed, which bottleneck mechanism appeared, which process caused it, and which application suffered.

Principles:

- **Never be the slowdown.** The always-on tier has a hard, published overhead budget.
- **Evidence, not guesses.** Every line of a diagnosis points to recorded events.
- **Say "unknown" before overclaiming.** No percentages that haven't been calibrated.
- **Observe → detect → explain → recommend.** No automatic changes to the system.
- **Local only by default.** Nothing leaves the machine unless the user opts in.
- **Accuracy is measured, not asserted.** A release ships only when it passes the evaluation suite.

## Target user and scope

The primary user is a technically capable developer who owns their machine: admin rights, no corporate management, running IDEs, Docker, WSL, browsers and local builds. Less technical users are a bonus, served later through a tray UI.

Scope is performance-first. "What's wrong with my machine" events (crashes, hangs, unexplained popups, hardware errors) land on the same incident timeline rather than in a separate product.

Non-goals:

- Not a Task Manager, htop or metrics dashboard replacement.
- Not a disk cleaner. Deciding which files are "useless" is out; low free space is detected as a mechanism only.
- No defragmenting. SSD maintenance config (TRIM, scheduled optimisation) is checked, never performed.
- No automatic remediation. v0.1 recommends only; user-approved actions come after accuracy is proven.
- Not a security or EDR tool, even though it records process lineage.
- No cloud and no telemetry upload.
- No LLM in v0.1.

## Questions Slowpoke answers

v0.1 answers the first three; the rest arrive by version.

| Question | Command | Version |
| --- | --- | --- |
| Why was the machine slow just now? | `slowpoke why` | v0.1 |
| What was that window that flashed? | `slowpoke spawned --visible` | v0.1 |
| What is Slowpoke itself costing me? | `slowpoke doctor` | v0.1 |
| What did the machine look like at 14:32? | `slowpoke rewind 14:32` | v0.2 |
| Does this keep happening? | `slowpoke blame` | v0.3 |
| What changed since yesterday? | `slowpoke diff yesterday today` | v0.3 |
| Why did this command take longer than usual? | `slowpoke run`, `slowpoke compare` | v0.4 |

## Architecture

Two services and a CLI: an elevated Rust collector turns OS signals into normalized events, and a Go core keeps state, detects incidents, attributes causes and serves the CLI. Rust says what happened; Go decides what it means.

&#91;embedded content: Slowpoke architecture · collector, core, CLI, storage, chaos tool\]

The collector is the only elevated part, and the CLI never sees raw events. The chaos tool drives the whole pipeline end to end.

- **Collector (Rust, elevated service).** One `Source` trait with a backend per OS selected by `cfg`: Windows uses ETW (ferrisetw) and performance counters; Linux uses procfs and PSI, later eBPF (aya); macOS polls via libproc and sysctl. A portable polling backend (`sysinfo` crate) gives every OS a basic tier. It holds a bounded in-memory ring buffer and does no analysis.
- **IPC.** Named pipe on Windows, Unix socket elsewhere. Access restricted to the core's service account and administrators. Compact binary encoding with a versioned schema.
- **Core (Go, service).** Ingest, per-process state, rolling windows, baselines, incident detection, rules, scoring, persistence in SQLite via a pure-Go driver (modernc.org/sqlite, no cgo), and a local API.
- **CLI (Go, unprivileged).** Talks to the core's local API only.

**Capability model.** At startup the collector announces which signals it supports. Each diagnosis rule declares the signals it needs. If any are missing, `why` says "can't diagnose X on this platform" instead of guessing.

**Privileges.** Kernel ETW and eBPF need admin or root, so the collector installs as a service. Unprivileged polling would miss short-lived processes, which rules out spawn forensics.

**Platform order.** Windows in v0.1, Linux in v0.2, macOS later or community-driven. The event schema is designed against both Windows and Linux concepts from day one. macOS will stay mostly polling: there is no ETW equivalent, and process-event monitoring needs an Apple entitlement.

**Build environment.** Develop in WSL2 and cross-compile Windows binaries (`GOOS=windows`; Rust `x86_64-pc-windows-msvc` via `cargo-xwin`). Run them on the host via WSL interop. WSL2 is a VM, so it suits Linux functional tests but not real-world diagnosis.

## Telemetry model

Telemetry runs in two tiers. The always-on tier stays inside a hard budget; the deep tier switches on only during a suspected incident. Accepted trade-off: the deep tier cannot see before it switches on, so an incident's onset is recorded at always-on detail only.

| Tier | What it records | When | Retention |
| --- | --- | --- | --- |
| Always-on | Process start/exit with parent, command line and session; per-process CPU and memory; system memory (available, commit, hard faults, paging I/O); disk throughput and latency; free space; AC/battery, power plan, CPU frequency; event-log symptoms | Events continuously, counters about every 1 s | In-memory ring (default 15 min); persisted only around incidents |
| Deep | Context switches and ready time; file I/O durations; per-I/O disk attribution; DNS timings; Defender engine events | On incident trigger, or user opt-in | Persisted with the incident |

Which signal lands in which tier is decided by the measurement spike (Roadmap, step 1), not by this table.

**Persistence.** Incidents with their evidence window (default 5 min before onset through the end), process lifecycle records, and aggregates for baselines. Everything else ages out of the ring.

**Overhead budget.** Targets to measure and publish, not claims:

| Measure | Target |
| --- | --- |
| Idle CPU, combined | < 0.5% |
| Average CPU, combined | < 1% |
| Memory, combined | < 75 MB |
| Disk writes, normal operation | < 1 MB/min |
| Event loss, normal operation | 0, with drop counters exposed |
| Queues | Always bounded |

**Self-exclusion.** Slowpoke's own processes and files are excluded from cause and victim lists, and its database directory is a documented candidate for antivirus exclusion.

## Signal catalog

Symptoms say what went wrong; mechanisms say why. An incident needs at least one of either, and a good diagnosis links a mechanism to a symptom.

**Symptoms**

| Symptom | Windows source | Linux source | Tier | Version |
| --- | --- | --- | --- | --- |
| UI hang | Win32k UI-delay events; Application Hang (1002) | Limited | Always-on | v0.1 |
| App crash | Application Error (1000); WER reports | journald, coredumps | Always-on | v0.1 |
| Short-lived or flashing window | Process start plus window/console creation | exec events | Always-on | v0.1 |
| User mark | `slowpoke mark` | `slowpoke mark` | n/a | v0.1 |
| Slow boot, standby or resume | Diagnostics-Performance log | systemd-analyze | Always-on | v0.2 |
| GPU driver reset | Display event 4101 (TDR) | dmesg | Always-on | v0.2 |
| Audio crackle, input lag | DPC/ISR latency | Limited | Deep | Later |
| Command slower than usual | `slowpoke run` | `slowpoke run` | n/a | v0.4 |

**Mechanisms**

| Mechanism | Windows source | Linux source | Tier | Version |
| --- | --- | --- | --- | --- |
| Memory pressure / paging | Available memory, commit charge, hard faults, paging I/O, compression store | PSI memory, vmstat | Always-on | v0.1 |
| CPU contention | Per-process CPU, processor queue length | PSI cpu | Always-on | v0.1 |
| Disk contention / latency | Disk latency counters | PSI io | Always-on | v0.1 |
| Low disk space | Volume free space | statfs | Always-on | v0.1 |
| Process spawn storm | Process start rate | exec rate | Always-on | v0.1 |
| Throttling / power changes | CPU frequency %, AC/battery, power plan, Kernel-Processor-Power events | cpufreq, thermal zones | Always-on | v0.1 |
| VM growth (WSL, Docker) | vmmem process memory | cgroups | Always-on | v0.1 |
| Scheduler delay | Context switch / ready-thread ETW | eBPF run-queue latency | Deep | v0.2 |
| Per-process disk attribution | DiskIo ETW | eBPF block I/O | Deep | v0.2 |
| Antivirus / indexer interference | Defender engine provider; FileIo durations | Limited | Deep | v0.2 |
| Network / DNS stalls | DNS client ETW; TCP retransmits | eBPF TCP | Deep | v0.2 |
| Hardware errors | WHEA events | EDAC, mcelog | Always-on | v0.2 |
| SSD maintenance config | TRIM and Optimize Drives status | fstrim timer | Config check | v0.2 |
| Driver / DPC latency | DPC/ISR ETW | Limited | Deep | Later |
| JVM GC pauses | JVM-specific | JVM-specific | Opt-in | Later |

WSL and Docker are visible only as one VM process from the host; activity inside them is opaque until a Linux-side collector runs there.

## Diagnosis engine

Deterministic rules run over a typed event timeline and produce ranked candidate causes, each tied to evidence. No ML and no LLM in v0.1.

**Incident lifecycle.**

1. A trigger fires: a symptom event (hang, crash, flashing window, user mark), or a mechanism crossing its threshold relative to baseline for a sustained period.
2. The incident opens, the evidence window is frozen from the ring, and the deep tier switches on.
3. The incident closes after a quiet period. Overlapping incidents merge.
4. Rules run over the frozen window; results are persisted with the evidence.

**Rule shape.** Each rule describes one mechanism chain: required signals, preconditions, temporal order, and output (mechanism, candidate cause, victims, evidence references). Example: process P starts → P's memory grows sharply within 10 s → available memory falls below threshold → hard faults exceed a multiple of baseline → a hang or latency symptom follows.

**Attribution.** Candidates are ranked on temporal precedence, size of the resource change, deviation from baseline, how much of the mechanism chain was observed, and recurrence. The score is for ranking only and is never shown as a probability.

**Showing confidence.** Three levels until scores are calibrated against labeled incidents:

- **Strong:** the full mechanism chain was observed.
- **Likely:** part of the chain was observed.
- **Unclear:** anomalies without a chain. Slowpoke says so plainly.

**Baselines.** Kept per metric and per process, keyed by context (AC vs battery, time-of-day bucket). For roughly the first three days, rules fall back to absolute thresholds and output flags "no baseline yet".

**Process identity.** A process is identified by name, image path and parent chain, not PID alone, so PID reuse doesn't corrupt history. Service hosts are split by service name. Long-lived daemons that do work for other processes (Gradle daemon, vmmem) are flagged as shared, so blame isn't pinned on the wrong caller.

**Spawn forensics.** This part is exact, not heuristic. For every process Slowpoke records parent and ancestor chain, originator (scheduled task, service, logon item, user shell), lifetime, and whether it created a visible window or console. "What flashed at 13:02?" becomes a lookup.

## CLI for v0.1

v0.1 ships ten commands; `why` and `spawned` are the product, the rest support them.

| Command | What it does |
| --- | --- |
| `slowpoke why` | Diagnoses the last auto-detected incident |
| `slowpoke why --at <time>` | Diagnoses the window around a given time |
| `slowpoke spawned [--since 10m] [--visible]` | Lists short-lived processes and what launched them |
| `slowpoke incidents` | Lists recent incidents |
| `slowpoke explain <id>` | Shows the raw evidence behind a diagnosis |
| `slowpoke mark "<note>"` | Records "something was wrong now" |
| `slowpoke label <id> --cause <x> --verdict correct\|partial\|wrong` | Records ground truth for an incident |
| `slowpoke export <id>` | Writes a redacted evidence bundle (for bug reports and fixtures) |
| `slowpoke status` | Current state and whether an incident is active |
| `slowpoke doctor` | Slowpoke's own CPU, memory, disk writes, event rate, drops |

Target shape of `why` output:

```
Incident #142  14:31:48 → 14:34:12  (2m 24s)

Symptom     idea64.exe UI unresponsive, 3 hangs (max 4.1s)
Mechanism   memory pressure → paging        confidence: strong
Cause       Docker Desktop startup (vmmem)

Timeline
14:31:42  docker.exe started (parent: explorer.exe)
14:31:47  vmmem memory +2.8 GB
14:31:51  available memory 6% (baseline 31%)
14:31:53  hard faults 6.2x baseline
14:32:03  idea64.exe UI hang 4.1s

Also present  chrome.exe 3.7 GB (stable, not rising)
Suggestion    cap WSL memory in .wslconfig (see `slowpoke explain 142`)
```

Deferred: `rewind` (v0.2), `blame` and `diff` (v0.3), `run` and `compare` (v0.4), `fix` (after accuracy is proven).

## Evaluation

Attribution accuracy is a release criterion: a version ships only when it meets its target on the chaos suite and the fixture set.

**Chaos tool.** A companion binary (working name `slowpoke-chaos`) that causes known problems and writes a ground-truth label for each run: scenario, injected process, start and end time. Scenarios are named after Slowpoke's moves, which are ordinary words and carry no trademark risk:

| Scenario | What it does | Expected diagnosis |
| --- | --- | --- |
| `amnesia` | Allocates and touches memory until the system pages | Memory pressure / paging, cause = injector |
| `yawn` | Spins CPU threads at high priority | CPU contention, cause = injector |
| `headbutt` | Hammers the disk with random reads and writes | Disk contention, cause = injector |
| `confusion` | Spawns a console window every N seconds | Flashing window, originator traced |
| `water-gun` | Churns small files in a directory Defender scans | Antivirus interference (v0.2) |
| `slack-off` | Switches to a power-saving plan to cut CPU frequency | Throttling / power change |
| `disable` | Fills a volume to near zero free space | Low disk space |

**Real incidents.** `slowpoke mark` records moments when something felt wrong, whether or not Slowpoke detected anything. `slowpoke label` records the actual cause after investigation, with a verdict on Slowpoke's diagnosis.

**Fixtures.** `slowpoke export` writes an incident's evidence window as a redacted bundle. The core can replay a bundle offline without the collector, so labeled bundles in `fixtures/` become regression tests for the rules. Contributors can submit their own bundles without touching collector code.

**Metrics.**

| Metric | Definition |
| --- | --- |
| Top-1 accuracy | Labeled cause ranked first |
| Top-3 accuracy | Labeled cause in the top three |
| Miss rate | Marks or chaos runs with no incident detected |
| False alarms per day | Incidents the user did not experience |
| Time to detect | Injection start to incident open |
| Overhead under load | `doctor` numbers during chaos runs |

**CI.** Fixture replay and rule tests run on every commit (Linux and Windows runners). The chaos suite runs on Windows runners, which have admin rights for ETW.

## Privacy and security

Slowpoke records sensitive metadata (command lines, paths, process names) and runs elevated, so both data and privilege are kept tightly scoped.

- **Local only.** No network calls. The optional LLM explainer (later) is opt-in, and an API-key mode shows a redacted preview before anything is sent.
- **Small, auditable collector.** The elevated part does collection only: no analysis, no network, no plugin loading. Releases are signed.
- **Locked-down IPC and storage.** The pipe/socket and the SQLite file are accessible only to the service account and administrators (or a dedicated `slowpoke` group). Otherwise any local process could read every other process's command lines through Slowpoke.
- **Command lines and paths are collected**, because spawn forensics needs them. A redaction setting masks arguments, and retention is configurable.
- **Export bundles are redacted by default:** usernames, home paths and command-line arguments are masked.

## Roadmap

v0.1 is ten steps. Step 1 is a gate: its numbers decide the telemetry tiers before any product code is written.

**v0.1 build steps (Windows)**

1. **Measurement spike.** Measure the CPU, memory and event rate of each ETW provider (kernel process, disk, memory, context switch, FileIo, Win32k, Defender) at idle and under a build. Output: which signals fit the always-on budget.
2. **Event schema v0.** Define normalized events and the capability manifest, checked against Linux equivalents.
3. **Collector skeleton.** Windows service, ETW process lifecycle plus counters, ring buffer, ACL-protected pipe, drop counters.
4. **Core skeleton.** Ingest, per-process state, rolling windows, SQLite, `status`, `doctor`.
5. **Spawn forensics.** Process lineage and originators, `spawned`. Validate against the cause of your old cmd-popup problem.
6. **Chaos tool v0.** `amnesia`, `yawn`, `headbutt`, `confusion`, each emitting ground-truth labels.
7. **Incidents and first rules.** Detector plus four rules (memory pressure/paging, CPU contention, disk contention, low disk space); `why`, `explain`, `incidents`.
8. **Fixtures and CI.** Export, offline replay, regression tests; record first top-1 and top-3 numbers.
9. **Dogfood.** Run it daily for 2–4 weeks, `mark` and `label` real incidents, tune rules and thresholds.
10. **Release.** Installer and uninstaller, signed binaries, README with measured overhead and accuracy numbers.

**Later versions**

- **v0.2:** `rewind`; Linux collector (procfs, PSI); deep tier (context switches, FileIo, Defender, DNS); boot/resume, WHEA and GPU-reset symptoms; SSD config checks.
- **v0.3:** Context-aware baselines, recurring-pattern detection, `blame`, `diff`.
- **v0.4:** `run` and `compare` with system-wide, daemon-aware attribution.
- **v0.5:** Tray icon and incident notifications; a curated recommendations catalog.
- **Later:** Opt-in LLM explainer, user-approved fixes once accuracy is proven, eBPF deep tier, macOS.

## Defaults and open decisions

These defaults were set without discussion; change any of them.

| Decision | Default |
| --- | --- |
| Ring buffer length | 15 min |
| Evidence window | 5 min before onset through incident end |
| Process lifecycle retention | 30 days |
| Baseline aggregate retention | 90 days |
| Event encoding | Protobuf (codegen for both languages, schema evolution) |
| License | Apache-2.0 |
| Data access | Administrators or a `slowpoke` group only |
| Notifications in v0.1 | None; CLI only |
| CPU frequency and power signals | In the v0.1 always-on tier (cheap, common laptop cause) |

Still open:

- [ ] **Name.** "Slowpoke" is a Pokémon trademark, and a Go key-value store on GitHub already uses the name. Fine as a codename; decide before promoting it widely.
- [ ] **Chaos tool name.** `slowpoke-chaos` is a placeholder.
- [ ] **v0.1 accuracy target.** Set after the first chaos-suite run in step 8.
- [ ] **`spawned` vs `why`.** Keep popup lookups as a separate command, or fold them into `why --at`.

## Prior art

No existing tool combines always-on recording, cause attribution and Windows support; each piece exists separately.

| Tool | What it is | What to borrow |
| --- | --- | --- |
| [below](https://github.com/facebookincubator/resctl) | Meta's Rust "time-traveling" resource monitor for Linux: record, replay, PSI | Replay UX and storage design for `rewind`; opinionated statistics |
| atop | Linux monitor that logs history as a daemon | Retention and replay conventions |
| [UIforETW](https://opensource.googleblog.com/2015/04/uiforetw-windows-profiling-made-easier.html) | ETW recording to an in-memory circular buffer, saved on a hotkey | Trigger model, provider choices, UI-delay detection |
| WPR / WPA | Microsoft's expert ETW recorder and analyzer | Wait analysis; optionally save an .etl per incident as an escape hatch |
| [Defender performance analyzer](https://learn.microsoft.com/en-us/defender-endpoint/tune-performance-defender-antivirus) | Records Defender scans; reports top files, extensions and processes | Provider and report shape for the antivirus mechanism; its caution on exclusions |
| Sysmon | Security-focused process, network and file event logging | Process lineage fields |
| [Netdata Anomaly Advisor](https://learn.netdata.cloud/docs/netdata-ai/troubleshooting/anomaly-advisor) | Per-metric anomaly detection ranked across all metrics | Ranking design for `diff`; its weakness with recurring patterns |
| Diagnostics-Performance log | Windows' built-in boot/standby degradation events naming a culprit | "Degradation + culprit" event schema |
| LatencyMon | DPC/ISR latency analysis | Driver-latency mechanism |
| Process Lasso | Automatic process priority intervention | Cautionary example for auto-remediation |
| [CrashSight Pro](https://github.com/unupunct/CrashSightPro) | Newer Windows stability dashboard with a similar "explain why" pitch | Positioning overlap to watch |

Methodology: Brendan Gregg's USE method and off-CPU analysis for the mechanism taxonomy. Bruce Dawson's Random ASCII blog as a case library of real Windows slowdowns to turn into chaos scenarios.
