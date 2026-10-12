# fleetguard

A Linux node agent and a safe-rollout controller, written in Go.

- **Agent**: reads host signals from `/proc` and PSI (`/proc/pressure/*`) and exposes CPU, memory, disk, network saturation and error metrics in Prometheus format.
- **Controller**: rolls a config change across a fleet in waves (canary first), gates each wave on agent metrics and SLO burn rate, pauses on a breach, and rolls back automatically.

## Status

Pre-design. The design doc (`docs/design.md`) comes first; no code until it is written.

## Development

Built with AI assistance (Claude). I can explain and modify every part of it.
