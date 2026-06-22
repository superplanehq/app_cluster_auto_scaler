# Agent instructions — Cluster Autoscaler

This document tells an AI agent everything it needs to know to work on
the Cluster Autoscaler SuperPlane app: what it does, how the graph is
wired, what each node assumes about its inputs, and the safe ways to
extend or debug it.

## What the app is

The app is a closed-loop autoscaler for a small Hetzner-backed worker
pool. It reacts to Grafana CPU alerts to add capacity, and runs a
periodic scale-down to remove capacity. Cluster membership is tracked
in canvas memory under the `cluster_nodes` namespace.

There is no scheduler/control-plane outside SuperPlane — the canvas
itself **is** the control loop.

## Topology

```
Grafana HighCPU Alert ─> Query Current Load ─> Read Cluster Nodes ─> Below Max (4)?
                                                                       ├─ true ─> Create Server ─> Bootstrap Worker ─┬─ success ─> Remember Node ─> Notify Scale Up
                                                                       │                                              └─ failed  ─> Rollback Failed Worker ─> Notify Bootstrap Failed
                                                                       └─ false ─> At Capacity (Slack)

Scale Down Check (every 10m) ─> Query Avg CPU (30m) ─> Read Cluster Nodes (Down) ─> Above Min (2)?
                                                                                       ├─ true ─> Delete Newest Node ─> Forget Node ─> Notify Scale Down
                                                                                       └─ false ─> (no-op)
```

## Node-by-node assumptions

| Node | Component | Notes for the agent |
|------|-----------|---------------------|
| `t-alert-up` | `grafana.onAlertFiring` | Filters on `alertNames: matches HighCPU`. Webhook is auto-bound by the SuperPlane Grafana integration; no contact-point setup needed in Grafana itself. |
| `sched-down` | `schedule` | Every 10 minutes, timezone UTC (`"0"`). Drives scale-down only. |
| `q-load-up`, `q-load-down` | `grafana.queryDataSource` | `dataSource: prometheus`. The result is consumed by Slack messages downstream using `data.results.A.frames[0].data.values[1][last]`. If you change the query, keep that path or update the Slack expressions. |
| `mem-count-up`, `mem-count-down` | `readMemory` | Both look up `namespace: cluster_nodes` matching `cluster=workers`. The `notFound` channel on `mem-count-up` is wired to `check-max` too so empty-cluster bootstrap works. Use `data.count` for size — it is always present (0 on notFound). |
| `check-max` | `if` | `data.count < 4`. Gates scale-up. |
| `check-min` | `if` | `data.count > 2`. Gates scale-down. |
| `create-server` | `hetzner.createServer` | `cpx11` in `fsn1`, `ubuntu-24.04`, sshKey `cluster-deploy`. cloud-init installs Docker on first boot. |
| `delete-server` | `hetzner.deleteServer` | Targets the **newest** node by reading the last entry of `Read Cluster Nodes (Down).data.values`. If you change scale-down policy (oldest first, biggest first), this is where it lives. |
| `rollback-delete` | `hetzner.deleteServer` | Targets `$['Create Server'].data.id` — only fires on the `failed` channel of `join-ssh`, so the just-created server is the only candidate. |
| `join-ssh` | `ssh` | Connects to `previous().data.publicIp` as `root` with the `cluster-deploy` secret. Commands are inline; the SSH runner executes them one line per `bash -c`, so multi-line constructs are forbidden — every statement must fit on one line. Connection retry is 20×15s. |
| `save-node`, `forget-node` | `upsertMemory`, `deleteMemory` | Keyed by `server_id`. `save-node` writes `server_id`, `server_name`, `ipv4`, `cluster`, `created_at`, `triggered_by` (uses `root().type` so it works for any trigger type). |
| `slack-up-ok`, `slack-down-ok`, `slack-up-cap`, `slack-bootstrap-failed` | `slack.sendTextMessage` | All target `#cluster-ops`. Use `root().type` — never `root().data.alerts[0]...`, that path only exists on Grafana-triggered runs and will error on any other entry. |

## Required integrations & secrets

The canvas does **not** ship with integration IDs. On install the user
must connect:

- A **Hetzner Cloud** integration
- A **Slack** integration (Bot scope: `chat:write` on the target channel)
- A **Grafana** integration (with permission to register webhooks and
  query datasources)

And create the secret:

- `cluster-deploy` with field `private-key` containing an OpenSSH
  private key whose matching public key is uploaded to Hetzner under
  the SSH-key name `cluster-deploy`.

## Conventions the agent must follow when editing

1. **Expressions** — use the `{{ ... }}` form. Reference upstream nodes
   as `$['Display Name'].data.field`, the trigger as `root().data...`,
   the immediate parent as `previous().data...`. Do not use `===`,
   `contains()`, or `output()` — those are not supported.
2. **Memory size** — always use `data.count` instead of
   `len(data.values)`. `count` is present in both `found` and
   `notFound` channels; `values` may be empty/missing.
3. **Trigger payload** — do not assume any specific trigger fired.
   `root().type` is always present (`grafana.alert.firing`,
   `scheduler.tick`, `manual.run`, …). Anything beyond `type` and
   `timestamp` should be guarded.
4. **SSH commands** — must be one statement per line, no `for`/`while`
   blocks across lines, no `\` line continuations. The runner executes
   each line in its own `bash -c`.
5. **Slack channels** — reference by `#name`, not numeric ID. Do not add
   the runtime `metadata.channel` block; SuperPlane fills it.
6. **Integration blocks** — when this template is generic, omit
   `integration: {id: ...}` entirely. The installer wires it after
   import.
7. **No personal data** — do not commit Slack channel IDs, integration
   UUIDs, Grafana webhook URLs, datasource UIDs, or Hetzner image
   numeric IDs. Use names/slugs (`#cluster-ops`, `prometheus`,
   `ubuntu-24.04`).
8. **Edges** — canonical fields are `sourceId`, `targetId`, `channel`.
   The parser rejects `source`/`target`/`from`/`to`.

## Common changes and how to make them

### Change cluster bounds

Edit the expressions in `check-max` (max) and `check-min` (min). The
display names also include the numbers — update both for clarity.

### Change server type/location

Edit `create-server.configuration.serverType` and `.location`. Verify
the combo is valid; if Hetzner returns
`422 location is not available for this server type`, pick another
location (`fsn1`, `hel1`, `ash`, `hil`, `sin`).

### Change workload

Replace the `docker pull` and `docker run` lines in `join-ssh.commands`.
Keep them on single lines.

### Add a new trigger source

Add the trigger node, then add an edge from it to `q-load-up` (for
scale-up) or `q-load-down` (for scale-down). The downstream graph is
trigger-agnostic — it relies only on `root().type` and the query
result, both of which are present for any trigger.

### Add a new notification target

Duplicate one of the Slack nodes, change the `component` to the new
vendor's send action (`discord.sendMessage`, `pagerduty.createIncident`,
etc.), and wire the same incoming edge.

## Debugging tips

- **Look at the live run** — open the failed run in the Monitor view.
  Each node shows the exact input payload it received. The most common
  failures are missing integration IDs (template state) and expression
  paths that don't exist on the actual payload.
- **`cannot fetch 0 from <nil>`** — an expression indexed into an array
  that didn't exist. Almost always a `root().data.alerts[0]...`
  reference being hit from a non-Grafana trigger. Switch to `root().type`.
- **SSH `unexpected end of file`** — a multi-line bash construct in
  `join-ssh.commands`. Collapse to one line.
- **SSH `docker.sock: no such file`** — the daemon hadn't started when
  the script ran. The wait loop at the top of `join-ssh.commands` should
  poll `docker info` (not `command -v docker`) up to 5 minutes.
- **Hetzner 422** — almost always a location/server-type mismatch.

## What to NOT touch without thinking

- The `mem-count-up` → `check-max` edge for the `notFound` channel.
  Removing it makes the first-ever scale-up impossible because empty
  memory wouldn't reach `check-max`.
- The `rollback-delete` failure path. Without it, a bootstrap failure
  leaves an orphaned Hetzner server billed by the hour.
- The `triggered_by` value `{{ root().type }}` in `save-node`. Hardcoding
  this to a Grafana-specific path will break any non-alert run.
