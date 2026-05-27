# Cantrip Workshop Environment Instructions

## Environment context

You are running inside a Workshop sandbox — an unprivileged LXD system
container with restricted host access. This is not a VM and not the host.

**Critical constraints:**

- No nested LXD, no host LXD socket, no Juju controller bootstrap inside the
  workshop. Local-controller workflows are unsafe here.
- Hardware (display, GPU, audio, USB) is unavailable unless an explicit plug
  has been connected.
- You cannot modify the workshop's container configuration.

## Persistent state

The SDK declares mount plugs that survive workshop refresh:

- `~/.config/cantrip` — Cantrip configuration, memory, transcripts
- `~/.local/share/cantrip` — Cantrip session DB, uv tool installs
- `~/.local/share/juju` — Juju client state (controllers, credentials)
- `~/.config/gh` — GitHub CLI auth and config

LLM provider keys (`ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, …) are **not**
persisted by the SDK. Pass them via `workshop exec --env` or set them in your
shell.

## Juju model

The workshop is **remote-controller-first**:

- Use `juju login` against an already-available controller.
- Run `juju status`, `juju deploy`, `juju refresh` etc. as a Juju **client**.
- Do not attempt to bootstrap a local controller — the workshop's interface
  model does not currently expose a first-class path for that.

## Resource-access protocol

Before using any hardware resource, follow this protocol:

1. **Detect** the resource via the indicators below.
2. **Validate** it is functional.
3. **Stop** if a required resource is missing.
4. **Report** to the user what is missing and why.
5. **Wait** for explicit permission or remediation before proceeding.

### Detection indicators

- **Display (X11):** `$DISPLAY` set, `/tmp/.X11-unix/` exists.
- **Display (Wayland):** `$WAYLAND_DISPLAY` set, socket present.
- **GPU:** `/dev/dri/` (Intel/AMD), `/dev/nvidia*` (NVIDIA).
- **Camera:** `/dev/video*` devices.
- **Audio:** `$PULSE_SERVER` set, or PipeWire socket in `$XDG_RUNTIME_DIR`.
- **USB:** `/dev/bus/usb/` or specific `/dev/*` nodes.

## Prohibited actions

- Never assume a resource is available without verification.
- Never silently fall back to a headless or stub implementation.
- Never improvise when required resources are missing.
- Never attempt to modify the workshop container configuration.
- Never attempt `juju bootstrap` inside the workshop.

Favour deterministic failure over silent degradation.
