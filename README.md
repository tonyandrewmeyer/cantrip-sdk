# Cantrip SDK for Workshop

This SDK provides [Cantrip](https://github.com/canonical/cantrip) — an
AI-powered autonomous agent that builds Juju charms — for AI-assisted charm
authoring within a Workshop. The agent is sandboxed in the workshop container.
Cantrip state, Juju client state, and GitHub CLI credentials persist between
workshop refreshes.

---

## Reference workshop

A minimal workshop:

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
sandboxed by the workshop, so it runs against the mounted project tree without
host access.

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
- **Juju:** run `juju login <controller>` inside the workshop. State persists
  via the `juju-config` plug.
- **GitHub:** run `gh auth login`. State persists via the `gh-config` plug.

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

### `juju-config`

- Interface: `mount`
- Workshop target: `/home/workshop/.local/share/juju`
- Purpose: Juju client state — controllers, credentials, models — across
  workshop refreshes.

### `gh-config`

- Interface: `mount`
- Workshop target: `/home/workshop/.config/gh`
- Purpose: GitHub CLI auth and configuration.

To mount your existing host state into the workshop, stop it first, remount,
then start it again:

```bash
workshop stop <workshop-name>
workshop remount <workshop-name>/cantrip:juju-config ~/.local/share/juju
workshop start <workshop-name>
```

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

- [Cantrip documentation](https://github.com/canonical/cantrip/tree/main/docs)
- [Workshop documentation](https://ubuntu.com/workshop/docs/)

---

## Community and support

- Workshop forum: [Discourse](https://discourse.ubuntu.com/)
- Please review our
  [Code of Conduct](https://ubuntu.com/community/ethos/code-of-conduct) before
  participating.

---

## Contributions

All contributions, including code, documentation updates, and issue reports,
are welcome.

---

## License and copyright

Copyright 2026 Canonical Ltd.

This program is free software: you can redistribute it and/or modify it under
the terms of the
[GNU Lesser General Public License version 2.1 (LGPLv2.1)](https://www.gnu.org/licenses/old-licenses/lgpl-2.1.html)
as published by the Free Software Foundation.
