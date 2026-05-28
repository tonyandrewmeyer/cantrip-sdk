# Cantrip SDK for Workshop

This SDK provides [Cantrip](https://tonyandrewmeyer.github.io/cantrip) — an
AI-powered autonomous agent that builds Juju charms — for AI-assisted charm
authoring within a Workshop. The agent is sandboxed in the workshop container.
Cantrip state, Juju client state, and GitHub CLI credentials persist between
workshop refreshes.

---

## Reference workshops

### Minimal — Cantrip only, no Juju

```yaml
# workshop.yaml
name: cantrip
base: ubuntu@24.04
sdks:
  - name: cantrip
    channel: latest/stable

actions:
  cantrip: cantrip "$@"
```

This creates a basic Cantrip charm-authoring environment. The agent is
sandboxed by the workshop, so it runs against the mounted project tree
(at `/project`) without host access.

### Authoring + remote Juju controller

A more realistic charm-authoring workshop composes Cantrip with the
`juju` SDK and binds its `controller` tunnel plug to a Juju controller
that lives on the host (or in a dedicated VM). Local controller
bootstrap is not supported by Workshop today — see the upstream Charm
Tech analysis under `docs/findings.md` of the
[`charm-tech-workshop`](https://github.com/canonical/charm-tech-workshop)
research repo.

```yaml
# workshop.yaml
name: cantrip-juju
base: ubuntu@24.04
sdks:
  # Expose the host's Juju controller API to the workshop via a tunnel slot.
  - name: system
    slots:
      controller-api:
        interface: tunnel
        endpoint: 17070
  - name: cantrip
  - name: juju

actions:
  bind: |
    # system-SDK tunnel slots are NOT auto-connected from `connections:`
    # (Workshop security policy). Run this once after `workshop launch`.
    workshop connect cantrip-juju/juju:controller cantrip-juju/system:controller-api
  cantrip: cantrip "$@"
```

Then, inside the workshop:

```bash
workshop run bind                # connect the tunnel
workshop shell
juju register <controller>       # or: juju login <controller>
cantrip
```

The Juju client state lives on the `juju-data` mount and survives
`workshop refresh`. Cantrip's own state (config, memory, sessions)
lives on the `cantrip-config` and `cantrip-data` mounts.

### Adding charm packaging

To `charmcraft pack` from inside the workshop, declare the `charmcraft`
SDK as well. Packing runs in `--destructive-mode` (i.e. directly in the
workshop, no nested build VM), so the workshop `base` **must match the
charm's `build-on` base** — multi-base charms need either the nested
`lxd` SDK (LXD-provider mode) or one workshop per base.

---

## Using the SDK

### Prerequisites, project layout

1. No prerequisite SDKs are required.
2. Place your charm project files in the project directory. Cantrip expects a
   Juju charm repo layout (`charmcraft.yaml`, `src/`, `tests/`, …) but works
   with greenfield directories too.
3. On launch, the SDK configures `PATH`, installs `juju-cantrip` via `uv tool
   install`, and provides a workshop-aware system prompt at
   `~/.config/cantrip/workshop-prompt.md`.

### Start a Cantrip session

Once the workshop is ready:

```bash
workshop shell
cantrip
```

This opens an interactive Cantrip session inside the workshop.

### Authenticate

- **LLM provider:** set `ANTHROPIC_API_KEY` or `GEMINI_API_KEY` (or any
  provider key Cantrip supports) inside the workshop. Pass it via
  `workshop run --env`, `workshop exec --env`, or with a tool such as
  [direnv](https://direnv.net/).
- **Juju:** if the workshop also declares the upstream `juju` SDK, run
  `juju login <controller>` inside; state persists via that SDK's
  `juju-data` mount.  Without the `juju` SDK there's no Juju client
  installed.
- **GitHub:** run `gh auth login` after each refresh — no SDK persists
  this today.

---

## Plugs (resources this SDK consumes)

### `cantrip-config`

- Interface: `mount`
- Workshop target: `/home/workshop/.config/cantrip`
- Purpose: Cantrip configuration, memory, transcripts, and the workshop
  system-prompt note.

### `cantrip-data`

- Interface: `mount`
- Workshop target: `/home/workshop/.local/share/cantrip`
- Purpose: Cantrip session SQLite DB and the persistent `uv tool` install
  directory.

> **Not declared by this SDK:** Juju client state (`~/.local/share/juju`)
> and GitHub CLI auth (`~/.config/gh`). Compose with the upstream `juju`
> SDK for Juju persistence — it ships a `juju-data` mount at the same
> path. There is no `gh` SDK in the Workshop catalogue today; `gh auth
> login` from inside the workshop after each refresh is the workaround.

## Slots (resources this SDK provides)

This SDK doesn't define any slots.

---

## Workshop boundary

The workshop is **remote-controller-first**. Inside the workshop you can:

- author and edit charm projects,
- run repo-local checks and unit tests,
- use Juju as a client against an already-available controller,
- use `git`, `gh`, and Cantrip's full tool surface.

You **cannot** (in v1):

- bootstrap a local Juju controller,
- run nested LXD,
- run `charmcraft pack` against a host-controlled LXD daemon
  (in-container packaging is being validated separately).

See `workshop-prompt.md` — it's also installed into the workshop for the
agent to consult.

---

## Documentation and guidance

- [Cantrip documentation](https://tonyandrewmeyer.github.io/cantrip/docs)
- [Cantrip source](https://github.com/tonyandrewmeyer/cantrip)
- [Workshop](https://ubuntu.com/workshop)

---

## Community and support

- Workshop forum: [Discourse](https://discourse.ubuntu.com/)
- Please review the [Code of Conduct](CODE_OF_CONDUCT.md) before participating.

---

## Contributions

All contributions, including code, documentation updates, and issue reports,
are welcome.
