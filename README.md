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

### Authoring against an existing host Juju controller (verified)

A realistic charm-authoring workshop composes Cantrip with the upstream
`juju` SDK and shares the host's Juju client state into the workshop.
Local controller bootstrap is **not** supported by Workshop today (see
the upstream Charm Tech analysis under `docs/findings.md` of the
`charm-tech-workshop` research repo).

```yaml
# workshop.yaml
name: cantrip-juju
base: ubuntu@24.04
sdks:
  - name: cantrip
  - name: juju

actions:
  juju-controllers: juju controllers
  juju-status: juju status -m <controller>:<model>
```

There are two ways to give the workshop access to the controller:

**Default LXD bridge — no tunnel needed.** If the host's Juju
controller is itself an LXD container (i.e. a `juju bootstrap
localhost` controller on the same machine), it's already reachable
from the workshop on the LXD bridge. Verified: `nc -zv <controller-ip>
17070` succeeds from inside the workshop with no `connect`
plumbing.

**Tunnel plug for off-bridge controllers.** If the controller lives
on a different host or on `localhost` (not on the LXD bridge), declare
a `system:` tunnel slot and bind the `juju` SDK's `controller` plug to
it. `system`-SDK tunnel slots are not auto-connected, so this needs a
manual `workshop connect` step (or wrap it in a workshop action) —
putting the bind in `connections:` is silently ignored.

After launch, share the host's Juju client state into the workshop so
controller registrations are visible:

```bash
workshop stop
workshop remount cantrip-juju/juju:juju-data ~/.local/share/juju
workshop start

# Verify the host controllers are visible inside:
workshop run -- juju-controllers
workshop run -- juju-status
```

The `juju-data` mount is owned by the upstream `juju` SDK (this SDK
deliberately does not declare a `juju-config` mount — see §7.3 of
`design/WORKSHOP_SDK.md` for the composition rationale).
Cantrip's own state (config, memory, sessions) lives on the
`cantrip-config` and `cantrip-data` mounts.

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
