# CLAUDE.md

## Repository

Desired state for one MySQL failover cluster reachable at
`my-ha.bigconfig.space`: three DigitalOcean droplets in Amsterdam running MySQL
8.0 as a single-primary Group Replication group, with snapshots and continuously
archived binary logs in the `mysql-ha-backup` R2 bucket and a daily verified
restore. Behaviour lives in `../mysql-ha`, which depends only on `../green`.

Tracked source is `colors.yml`, the copied `green` launcher, the
`.agents/skills/package-mysql-ha-green/` payload and its `skills-lock.json`,
a secret-free `.envrc`, the toolchain files, and documentation. `.colors/` is
generated and `.envrc.private` is secret. Never read or commit either.

## Commands

```sh
./green build              # render only; no credential, no provider
./green create --dry-run   # walk the graph; touch nothing
./green create             # converge, snapshot, verify a restore, assert
./green health             # read-only assertions against the live cluster
./green delete             # guarded teardown
```

`build` and `--dry-run` need no credentials. A real create or delete requires
explicit authorization. Keep `compute-prevent-destroy: true`; an authorized
delete uses a one-run `COLORS_PAR_COMPUTE_PREVENT_DESTROY=false` and never an
edit to the file. Never set `COLORS_PAR_PROFILE`.

## Coupling

The root launcher is a **copy** of `.agents/skills/package-mysql-ha-green/green`,
not a symlink. `npx skills update -p` rewrites the payload and leaves the root
file alone, so after every package update re-copy it and check they are
byte-identical:

```sh
npx skills update -p
cp .agents/skills/package-mysql-ha-green/green green
cmp green .agents/skills/package-mysql-ha-green/green
```

During development use `MYSQL_HA_LIB_ROOT=../mysql-ha`; a final run must use the
real pushed SHA in both copies.

## Operating notes

- Failover is automatic and needs no command. Group Replication elects the new
  primary; a ten-second timer on each member moves the reserved IP.
  `my-ha.bigconfig.space` is an `A` record pointing at that reserved IP and
  never changes.
- The reserved IP's assignment is deliberately **not** in OpenTofu state. Do not
  add `droplet_id` to `digitalocean_reserved_ip`.
- `./green create` is safe to re-run and converges an intact cluster. It also
  re-takes a snapshot and re-runs the verified restore every time, which takes a
  few minutes and is the point.
- The MySQL port is genuinely public. `digitalocean-client-sources` is the only
  thing keeping it narrow.

## Recovery

Full procedure in `../mysql-ha/README.md`. In short: `snapshot/latest.json`
names the newest dump, the dump sets `GTID_PURGED`, and the archived binary logs
under `binlog/<member>/` replay on top of it idempotently by GTID. Deleting the
cluster does not remove anything from the backup bucket.

## Git

Work on the current branch. Do not commit or push unless explicitly authorized.
