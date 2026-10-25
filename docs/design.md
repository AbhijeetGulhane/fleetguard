# fleetguard - design (v1)

Status: draft, 2026-10-25. Author: Abhijeet Gulhane (built with AI assistance).

## Problem
Pushing a config change to every host at once turns one bad change into a fleet-wide incident.
fleetguard rolls a change out in waves, watches host health on the nodes that already have it,
and stops and reverts on its own when health degrades.

## Goals
1. Agent: exposes memory, CPU, disk and TCP saturation/error signals from /proc and PSI in
   Prometheus format. Single static Linux binary.
2. Controller: rolls a versioned config through canary-first waves, gates each wave on agent
   health, pauses automatically on a breach and rolls back to the previous version.
3. Crash safety: a controller restarted mid-rollout resumes without re-applying or skipping nodes.
4. Agent overhead (CPU, RSS) measured and published, not estimated.

## Non-goals (v1)
SLO burn-rate gating; UI; cloud fleet / Terraform; controller HA; multi-tenancy; mTLS or RBAC;
FD/inode, conntrack and per-cgroup collectors; non-Linux platforms.

## Fleet
Three KVM VMs on WSL2 (rhel-lab + 2 clones), each with its own kernel, listed in a static
inventory file. Not capsule containers: they share the host kernel, so /proc/meminfo, /proc/stat,
/proc/diskstats and /proc/pressure are identical on every member and a canary-only fault would
be invisible.

## Components

### Agent (cmd/fleetguard-agent)
- Collectors (internal/collector), read on each scrape:
  - memory: /proc/meminfo, /proc/pressure/memory
  - cpu: /proc/stat, /proc/loadavg, /proc/pressure/cpu
  - disk: /proc/diskstats, /proc/pressure/io
  - tcp: /proc/net/snmp (RetransSegs, OutSegs), /proc/net/netstat (ListenOverflows)
- Exposes raw counters (Prometheus convention). Rates and deltas are computed by the consumer
  (the controller), so the delta math lives in one place.
- PSI files are discovered at startup. Absent PSI is a normal state (stock RHEL ships it off):
  fleetguard_psi_available{resource} = 0, never an error.
- Config endpoint: PUT /v1/config (bearer token; body + version + sha256). Idempotent: the same
  version is a no-op. Writes atomically (temp file, fsync, rename, fsync dir); keeps current +
  previous; runs an optional apply hook. GET /v1/config returns the applied version. Exposes
  fleetguard_config_version and fleetguard_config_apply_errors_total.
- Tests read fixture trees under testdata/ through an injectable proc root, never live /proc.

### Controller (cmd/fleetguard-controller)
- Inputs: inventory.yaml, rollout.yaml (target version, waves, bake time, gate rules), and a
  store of immutable configs/<version>.yaml.
- Waves: canary = 1 node, then cumulative percentages of the fleet (ceil, minimum 1 node).
- Per wave: persist intent -> PUT config to wave nodes (bounded concurrency, rate limit, retries
  with backoff) -> confirm applied version via GET -> bake: scrape every node that has the new
  version each interval for the bake duration -> gate.
  Pass: next wave. Fail: Paused -> RollingBack: re-push the previous version to every node that
  received the new one, newest wave first, and confirm.
- States: Pending -> Applying(w) -> Baking(w) -> ... -> Completed;
  on gate failure: Paused -> RollingBack -> RolledBack, or RollbackIncomplete (operator needed).
- Health gate: rules over the bake window. Rates come from counter deltas with reset detection
  (new < old means the agent restarted, not a negative rate). Initial rules, tuned in the demo:
  memory PSI some avg10 < 10%; TCP retransmit ratio (dRetransSegs/dOutSegs) < 2%;
  disk busy < 90%; agent reachable; config applied with zero apply errors.
  Missing metric or unreachable agent = fail closed. Rules are validated at load.
- State: one JSON file, written atomically before each action (write-ahead), plus flock on a
  lock file so only one controller runs. On start: load state, GET each node's applied version,
  reconcile, continue from the recorded step.
- Demo workload (cmd/fleetguard-demoload): reads the managed config (memory_mb, cpu_workers).
  A bad config drives memory pressure on the canary; this is what the demo rolls out.

## Failure modes
| Failure | Behavior |
|---|---|
| Controller crash mid-wave or mid-rollback | Intent persisted; restart reconciles applied versions and resumes. Idempotent PUT: no node skipped or double-applied. |
| Agent unreachable during apply | Retries with backoff; still failing -> wave fails -> rollback. |
| Agent down during bake | Gate fails closed. |
| Network partition (controller to some nodes) | Wave fails; unreachable nodes can't be rolled back -> RollbackIncomplete listing them; reconciled on next run. |
| Bad gate config | Validated at startup; controller refuses to start. |
| Bad config (the demo case) | Canary breaches the gate during bake -> pause + rollback before wave 2. |
| Agent restart mid-bake | Counter reset detected; that sample is dropped, not read as a negative rate. |
| PSI unavailable on a node | psi_available = 0; PSI rules on that node fail closed. |
| Second controller started | flock fails; second process exits. |
| Config write fails on a node (disk full) | Previous file untouched (atomic rename); apply error reported; wave fails. |
| State file unreadable | Refuse to start; never guess state. |

## Decisions
| # | Question | Choice | Why |
|---|---|---|---|
| 1 | Config delivery | Controller pushes (idempotent PUT) | Controller owns wave timing; simple reconcile. Pull is the right model at fleet scale (see below). |
| 2 | What the config is | Versioned immutable file; agent keeps current + previous | Rollback = re-push a known version. |
| 3 | How health is learned | Controller scrapes agent /metrics, parses text format | No Prometheus server needed in v1. |
| 4 | Gate SLI | n/a | SLO gating out of scope. |
| 5 | Inventory | Static YAML | 3 nodes; stable DHCP-reserved IPs. |
| 6 | State store | JSON + atomic rename + flock | Small state; no extra dependency. |
| 7 | Auth | Shared bearer token, plain HTTP, lab only | Documented limitation; mTLS later. |
| 8 | I/O PSI | Included in the disk collector | Present on both lab kernels; handled when absent. |
| 9 | Format, deps, lint, license | YAML (yaml.v3); client_golang, prometheus/common; golangci-lint; MIT | Minimal deps. |
| 10 | Go | 1.27.2 | Supported through the project window. |

## Layout
cmd/{fleetguard-agent,fleetguard-controller,fleetguard-demoload}
internal/{collector,agent,controller,gate,state}
testdata/proc/   docs/

## What changes at 60,000 nodes
Pull with jitter instead of push; waves follow failure domains (rack, datacenter), not
percentages; gates compare canary against a control cohort statistically instead of fixed
thresholds; metrics via a Prometheus tier, not direct scrapes; controller state in a replicated
store with leader election.

## Risks
- The controller is the hard part; start it right after agent v0.1.
- Thresholds on 3 nodes prove the mechanism, not statistical tuning.
- Token over plain HTTP is acceptable only on the lab network.
