# F5 BIG-IP VIP partition migration

Moves virtual servers (VIPs) listed in `vip_names`, and the objects they
depend on (pool, pool members, monitors, custom profiles), from
`source_partition` to `dest_partition`.

All migration parameters live in [`vars/migration.yml`](vars/migration.yml).
**Edit that file directly for every run** — see [How to run it](#how-to-run-it)
below. The playbook does support `-e` overrides, but `vars/migration.yml` is
the source of truth: it's what you read back later to know exactly what was
migrated, with what settings, and it survives across the multiple commands
each migration requires.

## How it works

BIG-IP enforces two different uniqueness constraints that are device-wide
(not per-partition), and they're what shape this playbook into two phases:

- **Node addresses** must be unique device-wide. Nodes are therefore never
  duplicated — pool members in `dest_partition` reference the existing node
  directly, wherever it actually lives (see [Known limitations](#known-limitations)
  for why "wherever it actually lives" matters).
- **VS destination addresses** must also be unique device-wide. A virtual
  server with a given destination IP/port cannot exist in two partitions at
  once, so unlike everything else, the VS can't be staged ahead of time.

**Phase 1 — staging (non-destructive, safe to re-run as often as you like):**

1. [`tasks/01_create_partition.yml`](tasks/01_create_partition.yml) — ensures `dest_partition` exists.
2. [`tasks/02_discover.yml`](tasks/02_discover.yml) — uses `bigip_device_info` to read the VIPs in
   `vip_names` (looking in both `source_partition` and `dest_partition`, so
   re-running after a cutover still works) and walks: VIP -> pool -> pool
   members -> nodes, VIP -> pool -> monitors, and VIP -> profiles.
3. [`tasks/03_migrate_monitors.yml`](tasks/03_migrate_monitors.yml), [`05_migrate_profiles.yml`](tasks/05_migrate_profiles.yml),
   [`06_migrate_pools.yml`](tasks/06_migrate_pools.yml) — recreate custom monitors, custom profiles,
   pools and pool members in `dest_partition`. The original VS in
   `source_partition` keeps serving traffic throughout this phase.

**Phase 2 — cutover (gated behind `cutover_virtual_servers: true`):**

4. [`tasks/07_cutover_virtual_servers.yml`](tasks/07_cutover_virtual_servers.yml) — removes the VS (and its
   virtual-address object) from `source_partition`, then immediately
   recreates it in `dest_partition` with the same destination IP/port. This
   is a brief, real interruption — there's no way to make the same
   destination IP exist in both partitions simultaneously.
5. [`tasks/08_cleanup_source.yml`](tasks/08_cleanup_source.yml) — only runs when
   `remove_source_after_migration: true` (which itself requires
   `cutover_virtual_servers: true`); deletes the now-unused pools, pool
   members, custom monitors and custom profiles left behind in
   `source_partition`. Nodes are never deleted.

### What gets cloned vs. referenced

| Object | Behavior |
|---|---|
| Virtual server | Recreated in `dest_partition` during the cutover (phase 2) |
| Pool | Recreated in `dest_partition` during staging (phase 1) |
| Pool members | Recreated in `dest_partition`; reference the node wherever it actually lives |
| Nodes | Never duplicated — always referenced from their real, existing location |
| Custom monitors (http, https, tcp, tcp\_half\_open, gateway\_icmp, icmp) | Cloned into `dest_partition` |
| Built-in monitors (`default_monitor_parents`, e.g. `http`, `tcp`) | Never cloned, always referenced as `/Common/<name>` |
| Custom profiles (client-ssl, server-ssl, http, tcp, udp, oneconnect) | Cloned into `dest_partition`, with the most commonly customized fields carried over |
| Built-in profiles (`default_profile_names`, e.g. `tcp`, `clientssl`, `f5-tcp-lan`) | Never cloned, always referenced as `/Common/<name>` |
| Other profile types (http2, FastL4, persistence, ...) / SNAT pools / LTM policies | Not duplicated; the recreated VS keeps referencing the original in `/Common` |

## How to run it

Every run starts the same way:

```bash
source .env   # loads BIGIP_HOST / BIGIP_USER / BIGIP_PASS into the shell
              # (ansible can't read the "export ..." .env file directly)
```

Edit [`vars/migration.yml`](vars/migration.yml) before each step below — set
`source_partition`, `dest_partition`, and `vip_names` to the VIPs you're
migrating in this run.

**Step 1 — stage pools/members/monitors/profiles.** Safe to re-run as many
times as you like; the original VS keeps serving traffic throughout.

```bash
# vars/migration.yml: cutover_virtual_servers: false, remove_source_after_migration: false
ansible-playbook playbooks/migrate_vips.yml
```

**Step 2 — review the staged objects** in `dest_partition` (GUI or `tmsh`),
then flip the cutover flag and re-run for the brief, real interruption that
recreates the VS itself:

```bash
# vars/migration.yml: cutover_virtual_servers: true
ansible-playbook playbooks/migrate_vips.yml
```

**Step 3 — once the cutover is confirmed good**, flip the cleanup flag too
and re-run to remove the now-unused objects from `source_partition`:

```bash
# vars/migration.yml: cutover_virtual_servers: true, remove_source_after_migration: true
ansible-playbook playbooks/migrate_vips.yml
```

### Chaining further migrations

To move the same VIP again later (e.g. `Common` -> `move-partition`, then
later `move-partition` -> `Sample_01`), edit `vars/migration.yml` so
`source_partition` is the partition it's in *now* (`move-partition`) and
`dest_partition` is where it's going next (`Sample_01`), reset both flags to
`false`, and repeat steps 1-3 above.

### One-off overrides

For a one-off run without editing the file (e.g. quick test), the same
variables can be passed with `-e`:

```bash
ansible-playbook playbooks/migrate_vips.yml \
  -e '{"vip_names": ["vs_app1", "vs_app2"]}' \
  -e cutover_virtual_servers=true
```

## Known limitations

- **Nodes are not duplicated, ever — including across chained migrations.**
  BIG-IP requires node addresses to be unique device-wide (not per
  partition) — attempting to create a node in `dest_partition` with the same
  address as an existing node elsewhere fails with `Invalid Node, the IP
  address ... already exists`. Pool members created in `dest_partition`
  therefore reference the node's *actual* existing partition, derived
  directly from the original member's `full_path` rather than assumed to be
  `source_partition`. This matters once you migrate the same VIP again later
  (e.g. `Common` -> `move-partition` -> `Sample_01`): the node may still be
  sitting in `Common` from the very first hop, not in the current run's
  `source_partition`. Nodes are never deleted by this playbook, regardless
  of which partition they're in.
- **VS cutover is a real, brief interruption.** The destination IP/port
  can't exist in both partitions at once, so phase 2 deletes the VS from
  `source_partition` before recreating it in `dest_partition`. Plan a
  maintenance window for this step; phase 1 has no such requirement.
- **Pool-level monitors** come from the pool's own `monitors` field when
  `bigip_device_info` returns it; if it's ever missing, the first pool
  member's monitor list is used as a stand-in. Verify this matches your
  intent for pools with per-member monitor overrides.
- **Built-in profiles/monitors** (see `default_profile_names` /
  `default_monitor_parents` in `vars/migration.yml`) are always referenced
  as `/Common/<name>` rather than cloned, since they're shared platform
  objects, not per-app config — even if a stale, same-named object happens
  to exist in some other partition from an earlier run (BIG-IP only allows a
  VS/pool to reference objects in `/Common` or its own partition, so this
  matters). If you find your environment uses other built-in profile/monitor
  names not already in these lists, add them — otherwise they'll get cloned
  needlessly into `dest_partition` instead of staying referenced from
  `/Common`.
- **Custom profiles**: only client-ssl, server-ssl, http, tcp, udp and
  oneconnect profile types are cloned, and only their most commonly
  customized fields are carried over (see comments in
  `tasks/05_migrate_profiles.yml`). Heavily customized profiles, or other
  profile types (e.g. http2, FastL4, persistence), are not duplicated; the
  recreated VS will keep referencing the original in `/Common` instead,
  which BIG-IP permits.
- **SNAT pools and LTM policies** referenced by a VS are not duplicated;
  the recreated VS keeps pointing at the original in `/Common`.
- Always review the objects staged in `dest_partition` before running
  with `cutover_virtual_servers: true`, and review the cutover result before
  running with `remove_source_after_migration: true`.
