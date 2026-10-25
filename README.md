# fleetguard

A Linux node agent and a safe-rollout controller, written in Go.

- **Agent**: reads host signals from `/proc` and PSI (`/proc/pressure/*`) and exposes CPU, memory, disk, network saturation and error metrics in Prometheus format.
- **Controller**: rolls a config change across a fleet in waves (canary first), gates each wave on agent health metrics, pauses on a breach, and rolls back automatically.

## Status

Design: [docs/design.md](docs/design.md). Implementation not started.

## Development

Built with AI assistance (Claude). I can explain and modify every part of it.

## License

MIT. See [LICENSE](LICENSE).
