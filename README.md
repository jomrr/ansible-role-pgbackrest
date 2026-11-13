# Ansible Role: pgbackrest

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-pgbackrest)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-pgbackrest)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-pgbackrest)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-pgbackrest/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-pgbackrest/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-pgbackrest/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-pgbackrest/actions/workflows/main.yml?query=branch%3Amain)

Install and configure pgBackRest for local and remote PostgreSQL backup and
restore.

## Purpose

Installs pgBackRest and configures local POSIX repositories, remote SSH
repositories and remote PostgreSQL hosts for backup and restore.

## Scope

### Managed

- pgBackRest, SSH client and cron packages on AlmaLinux, Debian, Fedora,
  openSUSE Leap, openSUSE Tumbleweed and Ubuntu.
- PGDG common RPM repository on AlmaLinux and Fedora, restricted to pgbackrest
  and its pgdg-libssh2 dependency, with signature verification.
- Signed openSUSE PostgreSQL project repository on Leap, with pgBackRest >= 2.55
  for PostgreSQL 18 support.
- Configuration, local repository directories, log/lock/spool directories and
  optional stanza initialization.
- Global and per-stanza backup/restore options and full, differential or
  incremental cron backup jobs.

### Not Managed

- PostgreSQL installation, database users, WAL archiving settings, service
  shutdown/startup and database deletion.
- Operating-system accounts, SSH server configuration, keys, host trust,
  firewall rules and TLS certificates.
- Automatic restore execution, automatic initial backups and deletion of
  repositories or retained backups.

## Requirements

- pgbackrest_user and pgbackrest_group must exist. On a database host use its
  PostgreSQL operating-system owner (postgres by default); create a dedicated
  account on a repository-only host.
- Remote SSH operation requires pgBackRest of the same version on both hosts and
  noninteractive SSH authentication with verified host keys. PostgreSQL hosts
  must reach the repository; repository-driven backups also require access from
  the repository to each database host.
- Online backups require a running PostgreSQL cluster with archive_mode=on and
  an archive_command invoking pgbackrest --stanza=NAME archive-push %p.
  Configure authentication so pgbackrest_user can connect to the cluster.
- On AlmaLinux and Fedora the role enables the official PGDG common repository
  for the target release and architecture. Leap uses the openSUSE PostgreSQL
  project repository for its release. Debian, Ubuntu and Tumbleweed use
  distribution packages. pgBackRest versions must match on connected hosts;
  coordinate package versions before using a mixed-distribution topology.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
  - name: ansible.posix
    version: '>=2.0.0'
  - name: community.crypto
    version: '>=3.0.0'
  - name: community.postgresql
    version: '>=3.12.0,<5.0.0'
```

## Role Variables

### `pgbackrest_user`

Type: `str`. Required: `false`.

Existing operating-system account running pgBackRest; use the PostgreSQL owner
on database hosts.

Default:

```yaml
pgbackrest_user: postgres
```

### `pgbackrest_group`

Type: `str`. Required: `false`.

Existing group allowed to read configuration and access local repositories.

Default:

```yaml
pgbackrest_group: postgres
```

### `pgbackrest_config_file`

Type: `path`. Required: `false`.

Absolute path of the managed pgBackRest configuration.

Default:

```yaml
pgbackrest_config_file: /etc/pgbackrest/pgbackrest.conf
```

### `pgbackrest_log_path`

Type: `path`. Required: `false`.

Directory for pgBackRest command logs.

Default:

```yaml
pgbackrest_log_path: /var/log/pgbackrest
```

### `pgbackrest_lock_path`

Type: `path`. Required: `false`.

Persistent directory for pgBackRest locks and stop files.

Default:

```yaml
pgbackrest_lock_path: /var/lib/pgbackrest/lock
```

### `pgbackrest_spool_path`

Type: `path`. Required: `false`.

Directory for asynchronous archive status and queues.

Default:

```yaml
pgbackrest_spool_path: /var/spool/pgbackrest
```

### `pgbackrest_repositories`

Type: `list`. Required: `false`.

Repository definitions; omit host for local storage and supply host for a remote
repository.

Default:

```yaml
pgbackrest_repositories:
  - id: 1
    path: /var/lib/pgbackrest/repo
    options:
      retention-full: 2
```

### `pgbackrest_options`

Type: `dict`. Required: `false`.

Additional global pgBackRest options using native names and YAML scalar or list
values.

Default:

```yaml
pgbackrest_options:
  log-level-console: info
  log-level-file: detail
  start-fast: true
```

### `pgbackrest_backup_options`

Type: `dict`. Required: `false`.

Native options in the global:backup section.

Default:

```yaml
pgbackrest_backup_options: {}
```

### `pgbackrest_restore_options`

Type: `dict`. Required: `false`.

Native options in the global:restore section, including recovery type and
target.

Default:

```yaml
pgbackrest_restore_options: {}
```

### `pgbackrest_stanzas`

Type: `list`. Required: `false`.

Database stanzas with native pg1-path, optional pg1-host and command-specific
options.

Default:

```yaml
pgbackrest_stanzas: []
```

### `pgbackrest_create_stanzas`

Type: `bool`. Required: `false`.

Create configured stanzas after PostgreSQL and remote access are ready.

Default:

```yaml
pgbackrest_create_stanzas: false
```

### `pgbackrest_schedules`

Type: `list`. Required: `false`.

Cron backup jobs; replacing the list removes previously managed jobs.

Default:

```yaml
pgbackrest_schedules: []
```

## Managed Files

- `/etc/pgbackrest/pgbackrest.conf` Default configuration path; root-owned with
  group read access and mode 0640.
- `/etc/cron.d/pgbackrest` Complete backup schedule; an empty schedule removes
  all managed jobs.
- `/etc/yum.repos.d/pgbackrest.repo` PGDG common repository on AlmaLinux and
  Fedora.
- `/etc/zypp/repos.d/pgbackrest-postgresql.repo` openSUSE PostgreSQL project
  repository on Leap.

## Check Mode

Reports configuration, directory and schedule changes using module check mode
support.

- Stanza creation does not run in check mode. Fresh hosts may require a real
  package installation before dependent resources can be checked.

## Service Behavior

pgBackRest runs on demand and reads its configuration for each command. Cron is
enabled and started when backup jobs are configured; PostgreSQL services are
managed separately.

## Security Notes

- Configuration rendering hides task output and diffs because options may
  contain repository encryption passphrases. Store secrets in Ansible Vault.
- The role does not disable SSH host-key verification. Provision known_hosts and
  appropriate authorized_keys before initializing remote stanzas.

## Operational Notes

- pgbackrest_repositories uses numeric IDs, absolute paths and optional host
  names. Only repositories without host create local directories. options
  contains native repository keys without the repoN- prefix (for example
  retention-full, host-user or cipher-pass). Storage is POSIX, local or on the
  remote repository host.
- pgbackrest_stanzas requires name and options.pg1-path. Use native
  pg1-host/pg1-host-user on a repository host to initiate remote database
  backups. Additional pg2-* settings support standby configurations.
- pgbackrest_options, pgbackrest_backup_options and pgbackrest_restore_options
  populate global, global:backup and global:restore. Per-stanza backup_options
  and restore_options override the corresponding global settings. Boolean values
  become y/n; lists in these sections produce repeated options (for example
  recovery-option). Use native option names valid for the installed pgBackRest
  version.
- There is no standalone offline configuration validator. pgbackrest check
  verifies the running database, repository and WAL archive after deployment; it
  is not suitable as a template validator. Run it before enabling production
  schedules.
- Set pgbackrest_create_stanzas=true only after PostgreSQL and all remote
  endpoints are ready. Repeated stanza-create calls report unchanged for
  existing valid stanzas. Backup jobs require initialized stanzas; schedules do
  not initialize them automatically.
- To back up locally or into a remote repository, run sudo -u postgres
  pgbackrest --stanza=main --repo=1 --type=full backup. On a dedicated
  repository host run the same command as that host’s pgbackrest_user.
- Restore always runs on the target database host. Stop its PostgreSQL service,
  provide an empty target directory, then run sudo -u postgres pgbackrest
  --stanza=main --repo=1 restore and start PostgreSQL. With a remote repository,
  use the same repo1-host connection as for backups. The role never stops
  PostgreSQL or removes database files.
- For point-in-time recovery set restore_options.type=time, target to the
  desired timestamp and target-action=promote; select a backup with --set when
  needed. Delta restores require an explicit delta=true option and may remove
  divergent files. Keep restore-only settings out of backup_options.
- With jomrr.postgresql configure archive_mode through
  postgresql_config_extra_restart and archive_command through
  postgresql_config_extra_reload; apply pgBackRest configuration before enabling
  WAL archiving. Use the same explicit --config path when overriding
  pgbackrest_config_file.
- Changing configuration or removing schedule entries is idempotent. Removing
  repository or stanza definitions does not delete backups. Local directories
  use mode 0750; their parent paths must allow access to pgbackrest_user.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Leap | latest | [jomrr/molecule-opensuse-leap:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-leap) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### Local backup and restore configuration

Apply to an existing PostgreSQL server, then initialize and check the stanza
before enabling schedules.

```yaml
---
- name: Configure local pgBackRest
  hosts: databases
  gather_facts: true
  roles:
    - role: jomrr.pgbackrest
      pgbackrest_stanzas:
        - name: main
          options:
            pg1-path: /var/lib/postgresql/17/main
          restore_options:
            process-max: 4
      pgbackrest_create_stanzas: true
      pgbackrest_schedules:
        - {stanza: main, type: full, weekday: '0', hour: '2'}
        - {stanza: main, type: diff, weekday: '1-6', hour: '2'}
```

### Database host with remote repository

Configure matching repository IDs and paths on both hosts. SSH credentials
and the repository account are provisioned separately.

```yaml
---
- name: Configure repository host
  hosts: backup_servers
  gather_facts: true
  roles:
    - role: jomrr.pgbackrest
      pgbackrest_user: pgbackrest
      pgbackrest_group: pgbackrest
      pgbackrest_repositories:
        - id: 1
          path: /srv/pgbackrest
          options:
            retention-full: 2
      pgbackrest_stanzas:
        - name: main
          options:
            pg1-host: db.example.net
            pg1-host-user: postgres
            pg1-path: /var/lib/postgresql/17/main

- name: Configure database host
  hosts: databases
  gather_facts: true
  roles:
    - role: jomrr.pgbackrest
      pgbackrest_repositories:
        - id: 1
          path: /srv/pgbackrest
          host: backup.example.net
          options:
            host-user: pgbackrest
            retention-full: 2
      pgbackrest_stanzas:
        - name: main
          options:
            pg1-path: /var/lib/postgresql/17/main
      pgbackrest_create_stanzas: true
```

### Point-in-time restore configuration

Apply these variables on the restore target, stop PostgreSQL, then
explicitly invoke pgbackrest restore. The timestamp is an example.

```yaml
pgbackrest_stanzas:
  - name: main
    options:
      pg1-path: /var/lib/postgresql/17/main
    restore_options:
      type: time
      target: '2026-11-01 12:00:00+00'
      target-action: promote
      process-max: 4
```

## References

- [pgBackRest user guide](https://pgbackrest.org/user-guide.html)
- [pgBackRest configuration reference](https://pgbackrest.org/configuration.html)
- [pgBackRest commands](https://pgbackrest.org/command.html)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
