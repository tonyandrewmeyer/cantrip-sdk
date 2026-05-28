# Cantrip Workshop Environment Instructions

## Environment context

You are running inside a Workshop sandbox — an unprivileged LXD system
container with restricted host access. This is not a VM and not the host.

**Critical constraints:**

- The project tree is mounted at `/project` (read-write; changes are
  shared with the host).
- `juju bootstrap localhost` and Canonical Kubernetes do **not** work
  inside a Workshop today (confirmed empirically — the controller
  machine is an LXD container at nesting depth 2 where snapd does not
  start). Local-controller workflows are unavailable.
- Nested LXD itself *does* work (`security.nesting=true` is the default)
  and is usable for `charmcraft pack` LXD-provider mode and for
  rockcraft — just not for hosting a Juju controller.
- Hardware (display, GPU, audio, USB) is unavailable unless an explicit
  plug has been connected.
- You cannot modify the workshop's container configuration.

## Persistent state

The SDK declares mount plugs that survive workshop refresh:

- `~/.config/cantrip` — Cantrip configuration, memory, transcripts
- `~/.local/share/cantrip` — Cantrip session DB, uv tool installs

Juju client state (`~/.local/share/juju`) is **not** mounted by this
SDK; if the workshop also declares the `juju` SDK it provides its own
`juju-data` mount at the same path. GitHub CLI state is not persisted
by any SDK today.

LLM provider keys (`ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, …) are **not**
persisted by the SDK. Pass them via `workshop exec --env` or set them in your
shell.

## Juju model

The workshop is **remote-controller-first**:

- Use `juju login` (or `juju register`) against an already-available
  controller. The controller typically lives on the host or in a
  dedicated VM and is reached via a tunnel plug.
- Run `juju status`, `juju deploy`, `juju refresh` etc. as a Juju
  **client**.
- Do not attempt `juju bootstrap localhost` — it will fail at the
  `juju-db` install step inside the nested LXD container.
- If the workshop declares a `juju:controller` tunnel plug to a
  `system:` tunnel slot, the binding is **not** auto-connected —
  `system`-SDK tunnel slots require a manual `workshop connect`
  (Workshop security policy). The workshop definition should expose
  this as a named action so the user can run it once after launch.

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
