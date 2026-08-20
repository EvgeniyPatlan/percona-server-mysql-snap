# Percona Server for MySQL Snap

[![Release to Snap Store](https://github.com/EvgeniyPatlan/percona-server-mysql-snap/actions/workflows/release.yaml/badge.svg)](https://github.com/EvgeniyPatlan/percona-server-mysql-snap/actions/workflows/release.yaml)

This repository contains the packaging metadata for creating a snap of
[Percona Server for MySQL](https://www.percona.com/mysql/software/percona-server-for-mysql),
built from the official Percona apt repositories
(`ps-84-lts` / `ps-97-lts` on repo.percona.com).
For more information on snaps, visit [snapcraft.io](https://snapcraft.io/).

Tracks:

| Track | Branch | Source repo |
|---|---|---|
| `8.4` | `8.4/edge` | https://repo.percona.com/ps-84-lts/apt |
| `9.7` | `9.7/edge` | https://repo.percona.com/ps-97-lts/apt |

## Installing the Snap

```bash
sudo snap install percona-server-mysql --channel 8.4/edge
```

The install hook initializes the data directory and configures the
`auth_socket` plugin for `root@localhost`, so right after installation:

```bash
sudo percona-server-mysql.mysql -u root
```

## RocksDB (MyRocks)

The `ha_rocksdb` storage engine is shipped but not enabled by default.
Enable it with Percona's admin helper:

```bash
sudo percona-server-mysql.ps-admin --enable-rocksdb -u root
```

## Telemetry

The Percona telemetry agent is not shipped in this snap, and the server-side
telemetry component is disabled by default
(`loose-percona_telemetry_disable = 1` in the snap's read-only defaults).

## Building the Snap

```bash
sudo snap install snapcraft
sudo snap install lxd
sudo lxd init --auto
snapcraft pack
sudo snap install ./percona-server-mysql_*.snap --dangerous --jailmode
```

## Testing the Snap

Using [Spread](https://github.com/canonical/spread):

```bash
snapcraft test                          # run all tests
ls -la spread/tests/                    # list all tests
snapcraft test -- spread/tests/smoke    # run one test suite
snapcraft test --debug                  # open shell on failure
```

## License

This snap packaging is free software, distributed under the Apache Software
License, version 2.0. See [LICENSE](LICENSE). Percona Server for MySQL is
distributed under the GPLv2 license (see `licenses/` inside the snap). This
repository's packaging is derived from
[canonical/mysql-snap](https://github.com/canonical/mysql-snap) (Apache-2.0).
