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

- **Node addresses** must be unique device-wide, and a pool member can only
  reference a node in `/Common` or its own partition — so a node can't just
  be referenced cross-partition, but it also can't be duplicated into
  `dest_partition` while the original still exists elsewhere. Nodes are
  therefore *moved*: deleted from wherever they currently live and
  recreated in `dest_partition`, same as the VS itself.
- **VS destination addresses** must also be unique device-wide. A virtual
  server with a given destination IP/port cannot exist in two partitions at
  once, so like nodes, the VS can't be staged ahead of time either.

**Phase 1 — staging (non-destructive, safe to re-run as often as you like):**

1. [`tasks/01_create_partition.yml`](tasks/01_create_partition.yml) — ensures `dest_partition` exists.
2. [`tasks/02_discover.yml`](tasks/02_discover.yml) — uses `bigip_device_info` to read the VIPs in
   `vip_names` (looking in both `source_partition` and `dest_partition`, so
   re-running after a cutover still works) and walks: VIP -> pool -> pool
   members -> nodes, VIP -> pool -> monitors, and VIP -> profiles.
3. [`tasks/03_migrate_monitors.yml`](tasks/03_migrate_monitors.yml), [`05_migrate_profiles.yml`](tasks/05_migrate_profiles.yml),
   [`06_migrate_pools.yml`](tasks/06_migrate_pools.yml) — recreate custom monitors, custom profiles,
   and the pools themselves (without members yet) in `dest_partition`. The
   original VS in `source_partition` keeps serving traffic throughout this
   phase.

**Phase 2 — cutover (gated behind `cutover_virtual_servers: true`):**

4. [`tasks/07_cutover_virtual_servers.yml`](tasks/07_cutover_virtual_servers.yml) — removes pool members and
   their backing nodes from `source_partition`, recreates the nodes and pool
   members in `dest_partition`, then removes the VS (and its virtual-address
   object) from `source_partition` and immediately recreates it in
   `dest_partition` with the same destination IP/port. This is a brief, real
   interruption — there's no way to make the same node address or
   destination IP exist in both partitions simultaneously.
5. [`tasks/08_cleanup_source.yml`](tasks/08_cleanup_source.yml) — only runs when
   `remove_source_after_migration: true` (which itself requires
   `cutover_virtual_servers: true`); deletes the now-empty pools, custom
   monitors and custom profiles left behind in `source_partition` (pool
   members, nodes and the VS are already gone by this point).

### What gets cloned vs. referenced

| Object | Behavior |
|---|---|
| Virtual server | Recreated in `dest_partition` during the cutover (phase 2) |
| Pool | Recreated in `dest_partition` during staging (phase 1) |
| Pool members | Recreated in `dest_partition` during the cutover (phase 2), once their backing node has moved |
| Nodes | Moved into `dest_partition` during the cutover (phase 2): deleted from their current partition, recreated with the same name/address |
| Custom monitors (http, https, tcp, tcp\_half\_open, gateway\_icmp, icmp) | Cloned into `dest_partition` |
| Built-in monitors (`default_monitor_parents`, e.g. `http`, `tcp`) | Never cloned, always referenced as `/Common/<name>` |
| Custom profiles (client-ssl, server-ssl, http, tcp, udp, oneconnect) | Cloned into `dest_partition`, with the most commonly customized fields carried over |
| Built-in profiles (`default_profile_names`, e.g. `tcp`, `clientssl`, `f5-tcp-lan`) | Never cloned, always referenced as `/Common/<name>` |
| Other profile types (http2, FastL4, persistence, ...) / SNAT pools / LTM policies | Not duplicated; the recreated VS keeps referencing the original in `/Common` |

## How to run it

First time only: copy [`.env.example`](.env.example) to `.env` and fill in
your real F5 host/credentials (`.env` is gitignored, never committed).

```bash
cp .env.example .env
$EDITOR .env
```

Every run starts the same way:

```bash
source .env   # loads BIGIP_HOST / BIGIP_USER / BIGIP_PASS into the shell
              # (ansible can't read the "export ..." .env file directly)
```

Edit [`vars/migration.yml`](vars/migration.yml) before each step below — set
`source_partition`, `dest_partition`, and `vip_names` to the VIPs you're
migrating in this run.

**Step 1 — stage pools/monitors/profiles.** Safe to re-run as many
times as you like; the original VS keeps serving traffic throughout (pool
members and nodes are not created yet - see [How it works](#how-it-works)).

```bash
# vars/migration.yml: cutover_virtual_servers: false, remove_source_after_migration: false
ansible-playbook playbooks/migrate_vips.yml
```

**Step 2 — review the staged objects** in `dest_partition` (GUI or `tmsh`),
then flip the cutover flag and re-run for the brief, real interruption that
moves the pool members/nodes and recreates the VS itself:

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

- **Nodes are moved, not duplicated — including across chained migrations.**
  BIG-IP requires node addresses to be unique device-wide (not per
  partition), and a pool member can only reference a node in `/Common` or
  its own partition. So during cutover, this playbook deletes each backing
  node from wherever it currently lives and recreates it (same name,
  address) in `dest_partition`, deriving its *actual* current partition
  directly from the original member's `full_path` rather than assuming
  `source_partition`. This matters once you migrate the same VIP again later
  (e.g. `Common` -> `move-partition` -> `Sample_01`): the node may still be
  sitting in `Common` from the very first hop, not in the current run's
  `source_partition`. If a node is shared by other pools/VIPs outside the
  ones listed in `vip_names` (in *any* partition, not just `source_partition`
  and `dest_partition`), moving it will affect those too — make sure nothing
  outside this migration's scope still needs the node where it currently
  lives before cutting over.
- **Cutover is a real, brief interruption.** Node addresses and VS
  destination IP/port both can't exist in two partitions at once, so phase 2
  deletes pool members + nodes, then the VS, from `source_partition` before
  recreating all of them in `dest_partition`. Plan a maintenance window for
  this step; phase 1 has no such requirement.
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

## v2: extended migration (folders, LTM policies, more)

[`playbooks/migrate_vips_v2.yml`](playbooks/migrate_vips_v2.yml) is a
parallel, independent entry point — it does not replace
`playbooks/migrate_vips.yml`, and the two share no task files (`tasks/v2/`
is a separate copy from `tasks/`). Use `v2` when your migration needs any
of the following, which the original playbook does not handle. Configure
it via [`vars/migration_v2.yml`](vars/migration_v2.yml) (same vars as
`vars/migration.yml`, plus the two below) and run it the same way:

```bash
source .env
ansible-playbook playbooks/migrate_vips_v2.yml
```

What `v2` adds on top of the original:

- **Custom LTM policies are migrated.** A VS's attached LTM (Local Traffic
  Manager) policy — if it's not already in `/Common` — is cloned into
  `dest_partition` (rules, conditions and actions included) via
  `tasks/v2/04_migrate_policies.yml`, and published automatically
  (`bigip_policy`'s `state: present` both creates and publishes — no manual
  GUI "publish" step needed). Already-shared `/Common` policies are
  referenced as-is, never cloned. Rule action/condition translation only
  covers the two shapes commonly seen in practice (a redirect action, and
  an exact-match `http_uri` condition) — anything else fails loudly rather
  than being silently mis-translated; extend
  `tasks/v2/04_migrate_policies.yml` if you hit an unrecognized shape.
  **True ASM/WAF security policies are a separate object model and are
  still not migrated** (`asm-policies` gather_subset, `bigip_asm_policy_*`
  modules) — that remains a manual step.
- **Pool monitors support the `m_of_n` ("min N of {...}") compound form**
  and `min-active-members`, not just simple AND-list monitors.
- **Virtual-address attributes are carried over explicitly**: `arp`,
  `icmp_echo`, `route_advertisement`, `traffic_group`, `netmask` — instead
  of relying on `bigip_virtual_server`'s auto-created default.
- **A few more VS-level fields are carried over explicitly**: `protocol`,
  `address_translation`, `port_translation`, `security_log_profiles`.
- **`node_target_partition`** (default: `dest_partition`, matching the
  original behavior) lets moved nodes land in `/Common` instead, so the
  same physical servers don't need to be re-moved on every future
  migration hop of the same VIP.
- **Folder support.** BIG-IP folders are per-object, not a single global
  setting — a VS/pool/policy can live in a sub-folder
  (`/partition/some_folder/object_name`) while its virtual-address and
  nodes stay flat at the partition root, and `v2` derives each object's
  folder from its own discovered path rather than a configured folder
  name. `preserve_folders: true` (the default) recreates objects in the
  same-named sub-folder of `dest_partition`; set it to `false` to flatten
  everything to the partition root instead. There is no dedicated folder
  module in the `f5networks.f5_modules` collection, so folder creation
  uses `bigip_command`/tmsh directly.

### v2 known limitations (in addition to the ones above)

- `icmp_echo`/`route_advertisement` values other than enabled/disabled
  (e.g. `always`, `selective`, `any`, `all`) don't survive the discovery
  round-trip — `bigip_device_info` collapses them to `None`. This is a
  structural limitation of the installed F5 collection's fact gathering,
  not something fixable without bypassing `bigip_device_info` entirely.
  Verify/correct these manually post-migration if your original used a
  non-enabled/disabled value.
- VS `mask` is not carried over — `bigip_device_info` doesn't expose it
  for virtual servers at all. The recreated VS relies on
  `bigip_virtual_server`'s default (`255.255.255.255` for IPv4 host
  destinations).
- `vip_names` are always bare leaf names regardless of folder — if two
  VIPs share the same bare name in different folders of the same
  partition, this playbook can't disambiguate them.
- True ASM/WAF security policies are not migrated (see above).
