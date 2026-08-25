# percona-server-mysql

Percona Server for MySQL packaged as a strict-confinement snap: the
`percona-server-server` and `percona-server-client` packages plus the
`percona-server-rocksdb` (MyRocks) storage engine, staged unmodified from
Percona's official apt repository at `repo.percona.com` — nothing is
compiled from source. Two major versions are published as separate
branches/tracks, each pinned to an exact upstream package version (see
below). Base: `core26`.

## Why this snap

Installing this snap gets the whole Percona Server for MySQL stack —
server, client, and MyRocks — in one artifact, with every package pinned to
an exact upstream version rather than "whatever is latest on 8.4". The
install hook runs `mysqld --initialize`, switches `root@localhost` to
`auth_socket` authentication, and starts the server automatically, so
there is no separate bootstrap step. Each supported major (8.4, 9.7) is a
distinct branch/track, so moving to a new major is an explicit channel
switch rather than something the snap decides for you on refresh.

## Tracks and branches

| Branch | apt source | Version |
|---|---|---|
| `8.4/edge` | `repo.percona.com/ps-84-lts/apt` (resolute, main) | 8.4.11-11 |
| `9.7/edge` | `repo.percona.com/ps-97-lts/apt` (resolute, main) | 9.7.1-1 |

## Getting the snap

### From a CI build

Every push to a `*/edge` branch, every pull request, and every manual
`workflow_dispatch` run of the `Tests` workflow builds the snap (amd64 and
arm64) and runs the full spread suite against it.

1. Open the workflow run in GitHub Actions and download the
   `snap-packages` artifact.
2. Unzip it.
3. Install:
   ```
   sudo snap install ./percona-server-mysql_<version>_amd64.snap --dangerous --jailmode
   ```
   (substitute the `arm64` filename on that architecture).

### From source

```
git clone https://github.com/EvgeniyPatlan/percona-server-mysql-snap.git
cd percona-server-mysql-snap
git checkout 8.4/edge   # or 9.7/edge
snapcraft pack
sudo snap install ./percona-server-mysql_*.snap --dangerous --jailmode
```

Requires the `snapcraft` and `lxd` snaps.

Store channels exist for each track (`8.4/edge`, `9.7/edge`), but the
release workflow only publishes when the repository's `RELEASE_ENABLED`
variable is set, so Store availability isn't guaranteed.

## First steps

`mysqld` starts automatically on install. The install hook switches
`root@localhost` to `auth_socket` authentication, so connect locally
without a password:

```
sudo percona-server-mysql.mysql -u root
```

A TCP connection as `root@localhost` is denied by design — `auth_socket`
only accepts the local Unix socket — so create a dedicated user with a
password for TCP/network access.

## Services and apps

| App | Kind | Purpose |
|---|---|---|
| `mysqld` | daemon, auto-started | MySQL server, run under a supervisor loop that re-execs on the SQL `RESTART` statement (exit code 16) |
| `mysql` | CLI | interactive/batch SQL client |
| `mysqladmin` | CLI | server administration (ping, status, shutdown, …) |
| `mysqlcheck` | CLI | table check/repair/analyze/optimize |
| `mysqldump` | CLI | logical backup |
| `mysqlimport` | CLI | load delimited text files |
| `mysqlshow` | CLI | list databases/tables/columns |
| `mysqlslap` | CLI | load-testing/benchmark tool |
| `ps-admin` | CLI | Percona Server admin helper (e.g. enabling RocksDB) |

`mysqld` is the only daemon and it is enabled by default:

```
sudo snap stop percona-server-mysql.mysqld
sudo snap start percona-server-mysql.mysqld
sudo snap restart percona-server-mysql.mysqld
```

This snap has no `snap set` configuration knobs — all server tuning goes
through the config file below.

## Configuration and data paths

| Item | Path |
|---|---|
| Read-only defaults | `/snap/percona-server-mysql/current/etc/my.cnf` (`!includedir` pulls in the directory below) |
| Editable config | `/var/snap/percona-server-mysql/current/etc/mysqld.cnf` |
| Data directory | `/var/snap/percona-server-mysql/common/data` (survives snap refreshes) |
| Error log | `/var/snap/percona-server-mysql/current/log/error.log` |
| Slow / general / binlog logs | `/var/snap/percona-server-mysql/current/log/{mysql-slow,query,mysql-bin}.log` (disabled by default; uncomment the relevant lines in `mysqld.cnf`) |
| Socket | `/var/snap/percona-server-mysql/current/run/mysqld.sock` |
| X Protocol socket | `/var/snap/percona-server-mysql/current/run/mysqlx.sock` (port 33060) |

## Enabling the RocksDB (MyRocks) storage engine

```
sudo percona-server-mysql.ps-admin --enable-rocksdb -u root
percona-server-mysql.mysql -u root -e "SHOW ENGINES;" | grep -i rocksdb
percona-server-mysql.mysql -u root -e "CREATE TABLE t (id INT PRIMARY KEY, v VARCHAR(50)) ENGINE=ROCKSDB;"
```

The engine registration persists across `snap restart
percona-server-mysql.mysqld`.

## Testing

Every push and pull request runs the full spread suite against a real
snapd install inside an LXD `ubuntu-24.04` VM, on both `amd64` and
`arm64`. Suites: `aliases`, `cli_mysqladmin`, `cli_mysqlcheck`,
`cli_mysqlcli`, `cli_mysqldump`, `cli_mysqlimport`, `cli_mysqlshow`,
`cli_mysqlslap`, `daemon_mysqld`, `rocksdb`, `smoke`, `storage`, and
`upgrade` (currently marked `manual` until the snap is published to
`8.4/edge`).

To reproduce locally:

```
snapcraft pack
CRAFT_ARTIFACT=$(pwd)/percona-server-mysql_<version>_amd64.snap spread -v
```

(`spread` from `go install github.com/canonical/spread/cmd/spread@latest`;
needs the `lxd` snap.)

## Updating to a new Percona release

`scripts/bump-version.sh` checks every exact-pinned package in
`snap/snapcraft.yaml` against the apt indexes declared under
`package-repositories`, and bumps any pin (and the top-level `version:`
field, derived from the `percona-server-server` pin) that is out of date.

Each track declares two apt repositories: the track's own Percona repo
(e.g. `repo.percona.com/ps-84-lts/apt` for `8.4/edge`), which is where
`percona-server-server`, `percona-server-client`, and
`percona-server-rocksdb` are all pinned from, plus
`repo.percona.com/telemetry/apt`, needed only because
`percona-server-server` depends on `percona-telemetry-agent` at the apt
level (the agent itself is pruned from the packaged snap and is never
exact-pinned, so it is never a bump target).

### Automated

The `Update check` workflow (`.github/workflows/update-check.yaml`) runs
weekly and, for each `*/edge` branch (`8.4/edge`, `9.7/edge`), runs the
same script and opens a pull request per branch that has an available
update. The PR:

- touches only `snap/snapcraft.yaml`, with the pin diff as the commit;
- contains the script's summary table (old/new version per package) in its
  description;
- is verified the same way any other PR is: CI (`Tests`) builds the snap for
  `amd64` and `arm64` and runs the full spread suite against it. Merging the
  PR into its track branch produces the downloadable `snap-packages`
  artifact described above.

To trigger an immediate check instead of waiting for the weekly run, start
the `Update check` workflow manually from the Actions tab (`workflow_dispatch`,
optionally scoped to one branch via the `branch` input).

If a bump PR is closed without merging, its `bump/<track>-<version>` branch
is left behind and that exact version is skipped on every future run until
the branch is deleted (or a newer version ships) — delete the branch if you
want the check retried for that version.

### Manual

```
./scripts/bump-version.sh
git diff
```

Review the diff, then commit and push as usual.

### Scope

The script only updates pins within the current track (`8.4` or `9.7`). A
new Percona major version means a new track/branch and, per the Percona
publishing model for Percona Server for MySQL, a new apt repository path
(`repo.percona.com/ps-<NN>-lts/apt`) — that's a manual, one-time setup, not
something this script does.

## License

The snap packaging is Apache-2.0. Upstream component licenses (Percona
Server for MySQL, its client, and MyRocks) are shipped under `licenses/`
inside the snap.
