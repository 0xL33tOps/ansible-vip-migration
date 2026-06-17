# F5 BIG-IP VIP partition migration

Moves virtual servers (VIPs) listed in `vip_names`, and the objects they
depend on (pool, pool members, monitors, custom profiles), from
`source_partition` (default `Common`) to `dest_partition` (default
`move-partition`).

This is a two-phase process, because BIG-IP enforces two different
device-wide (not per-partition) uniqueness constraints that shape what can
be staged ahead of time versus what requires a cutover:

- **Node addresses** must be unique device-wide. Nodes are therefore never
  duplicated — pool members in `dest_partition` reference the existing node
  in `source_partition` directly.
- **VS destination addresses** must also be unique device-wide. A virtual
  server with the same destination IP/port cannot exist in both partitions
  at once, so unlike everything else, it can't be staged in advance.

## How it works

**Phase 1 — staging (non-destructive, safe to re-run):**

1. `tasks/01_create_partition.yml` — ensures the destination partition exists.
2. `tasks/02_discover.yml` — uses `bigip_device_info` to read the VIPs and
   walks: VIP -> pool -> pool members -> nodes, and VIP -> pool members ->
   monitors, and VIP -> profiles.
3. `tasks/03_migrate_monitors.yml`, `05_migrate_profiles.yml`,
   `06_migrate_pools.yml` — recreate monitors, custom profiles, pools and
   pool members in `dest_partition`. The original VS in `source_partition`
   keeps serving traffic throughout this phase.

**Phase 2 — cutover (gated behind `cutover_virtual_servers: true`):**

4. `tasks/07_cutover_virtual_servers.yml` — removes the VS (and its
   virtual-address object) from `source_partition`, then immediately
   recreates it in `dest_partition` with the same destination IP/port. This
   is a brief, real interruption — there's no way to make the same
   destination IP exist in both partitions simultaneously.
5. `tasks/08_cleanup_source.yml` — only runs when
   `remove_source_after_migration: true` (which itself requires
   `cutover_virtual_servers: true`); deletes the now-unused pools, pool
   members, monitors and custom profiles left behind in `source_partition`.
   Nodes are never deleted, since the migrated pool members still depend on
   them.

## Usage

```bash
source .env   # loads BIGIP_HOST / BIGIP_USER / BIGIP_PASS into the shell
              # (ansible can't read the "export ..." .env file directly)

# Edit vars/migration.yml to list the VIPs you want to move, or override inline.
# Phase 1: stage pools/members/monitors/profiles. Repeatable, no VS impact.
ansible-playbook playbooks/migrate_vips.yml -e '{"vip_names": ["vs_app1", "vs_app2"]}'

# Review the staged objects in /move-partition. When ready for the cutover:
ansible-playbook playbooks/migrate_vips.yml -e cutover_virtual_servers=true \
  -e '{"vip_names": ["vs_app1", "vs_app2"]}'

# Once the cutover is confirmed good, clean up the source partition:
ansible-playbook playbooks/migrate_vips.yml -e cutover_virtual_servers=true \
  -e remove_source_after_migration=true -e '{"vip_names": ["vs_app1", "vs_app2"]}'
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
- **Pool-level monitors**: `bigip_device_info` does not expose the health
  monitor(s) assigned directly to a pool, only the monitors in effect on
  each pool member. The first member's monitor list is used as a stand-in
  for the pool's monitor assignment — verify this matches your intent for
  pools with per-member monitor overrides.
- **Built-in profiles/monitors** (`tcp`, `http`, `clientssl`, etc., see
  `default_profile_names` / `default_monitor_parents` in
  `vars/migration.yml`) are left referenced from `/Common` rather than
  cloned, since they're shared platform objects, not per-app config.
- **Custom profiles**: only client-ssl, server-ssl, http, tcp, udp and
  oneconnect profile types are cloned, and only their most commonly
  customized fields are carried over (see comments in
  `tasks/05_migrate_profiles.yml`). Heavily customized profiles, or other
  profile types (e.g. http2, FastL4, persistence), are not duplicated; the
  recreated VIP will keep referencing the original in `/Common` instead,
  which BIG-IP permits.
- **SNAT pools and LTM policies** referenced by a VIP are not duplicated;
  the recreated VIP keeps pointing at the original in `/Common`.
- Always review the objects staged in `/move-partition` before running
  with `cutover_virtual_servers: true`, and review the cutover result before
  running with `remove_source_after_migration: true`.
