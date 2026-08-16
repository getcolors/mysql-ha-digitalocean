# mysql-ha-digitalocean

Desired state for `my-ha.bigconfig.space`: a three-member **MySQL Group
Replication** cluster on `s-2vcpu-4gb` DigitalOcean droplets in `ams3`, inside
that region's default VPC, with daily snapshots and continuously archived binary
logs in the `mysql-ha-backup` Cloudflare R2 bucket and a restore that is
actually performed and asserted every day.

```sh
./green build
./green create --dry-run
./green create
./green health
```

The cluster is its own quorum: three mysqld processes form the Paxos group, so
there is no orchestrator, no etcd and no fourth machine. Clients connect to
`my-ha.bigconfig.space`, an `A` record whose content is a DigitalOcean reserved
IP. Failover moves that address to the new primary within about ten seconds of
the election; the record itself never changes.

Credentials live only in the gitignored `.envrc.private`, as the `COLORS_PAR_*`
variables listed at the top of `colors.yml`. The database needs exactly two of
them — the admin password and the replication password. Never set
`COLORS_PAR_PROFILE`.

Behaviour lives in [`getcolors/mysql-ha`](https://github.com/getcolors/mysql-ha);
this repository holds no source code. Deleting the cluster is protected by
`compute-prevent-destroy: true` and removes nothing from the backup bucket.
