Ansible Role: one_mind.sentry
=============================

Installs and upgrades [Sentry self-hosted](https://github.com/getsentry/self-hosted) with Docker Compose.

What the role does
------------------

1. Checks out `getsentry/self-hosted` at the `sentry_version` tag into `sentry_docker_compose_project_folder` (`force: true`, local changes to tracked files are discarded).
2. Sets `.env` keys from `sentry_env` and `SENTRY_BIND`.
3. Runs `./install.sh --no-report-self-hosted-issues --skip-user-creation` as an async task limited by `sentry_install_async`.
4. Replaces `docker-compose.yml` with the role template: the upstream compose file of the same tag plus `deploy.resources.limits` from `sentry_limits` and ClickHouse `MAX_MEMORY_USAGE_RATIO`.
5. Edits `sentry/config.yml` (`sentry_mail`, `sentry_url`, `sentry_config`) and `sentry/sentry.conf.py` (`sentry_config_py`, `sentry_config_py_lines`, `sentry_config_py_absent`).
6. Starts the compose project and creates or updates `sentry_superusers`.

Requirements
------------

* Docker with the compose v2 plugin
* `community.docker` collection (`docker_compose_v2` module)

Role Variables
--------------

| Variable | Default | Description |
|---|---|---|
| `sentry_version` | `26.10.1` | self-hosted tag. The template matches this tag only, see [Upgrading](#upgrading) |
| `sentry_docker_compose_project_folder` | `/opt/docker/sentry` | checkout directory |
| `sentry_bind` | `9000` | `SENTRY_BIND` in `.env` |
| `sentry_install_async` | `3600` | timeout in seconds for `install.sh` and `compose up`. PostgreSQL major upgrades inside `install.sh` need more than 600 |
| `sentry_env` | `{}` | extra `.env` keys |
| `sentry_mail` | `{}` | `mail.*` keys in `sentry/config.yml` |
| `sentry_url` | `https://url.com` | `system.url-prefix`; set to `null` to keep the file value |
| `sentry_config` | `{}` | other keys in `sentry/config.yml` |
| `sentry_config_py` | `{}` | `key: value` lines inside dicts of `sentry.conf.py`, matched by key |
| `sentry_config_py_lines` | `[]` | list of `{regexp, line}`: the line matching `regexp` is replaced, otherwise `line` is appended to the end of `sentry.conf.py` |
| `sentry_config_py_absent` | `[]` | regexps of lines to remove from `sentry.conf.py` |
| `sentry_superusers` | `[]` | list of `{email, password}`, created with `--force-update` |
| `sentry_apply_automatic_config_updates` | `"0"` | `APPLY_AUTOMATIC_CONFIG_UPDATES` for `install.sh`: `"1"` lets it patch `sentry.conf.py` when upstream adds new infrastructure |
| `sentry_clickhouse_max_memory_usage_ratio` | `0.3` | ClickHouse `MAX_MEMORY_USAGE_RATIO` |
| `sentry_limits` | see `defaults/main.yml` | `cpu` and `ram` limits per compose service. Every service that has a `deploy` block in the template needs a key |

Example
-------

```yaml
sentry_docker_compose_project_folder: /etc/sentry
sentry_version: 26.10.1
sentry_bind: 9000

sentry_mail:
  host: mail.example.com
  from: sentry@example.com
  port: 587
  username: sentry@example.com
  password: "{{ vault_sentry_mail_password }}"

sentry_url: https://sentry.example.com
sentry_superusers:
  - email: admin@example.com
    password: "{{ vault_sentry_superuser_password }}"

sentry_apply_automatic_config_updates: "1"
sentry_config_py_lines:
  - regexp: '^from sentry\.conf\.types\.uptime import UptimeRegionConfig$'
    line: 'from sentry.conf.types.uptime import UptimeRegionConfig'
  - regexp: '^UPTIME_REGIONS = '
    line: 'UPTIME_REGIONS = [UptimeRegionConfig(slug="default", name="Default Region", config_redis_cluster=SENTRY_UPTIME_DETECTOR_CLUSTER, api_endpoint="uptime-checker:12345")]'
sentry_config_py_absent:
  - '^\s*"organizations:incidents",\s*$'
```

`install.sh` creates `sentry.conf.py` from the example only on the first install. Installations created before 26.10.1 do not get the `UPTIME_REGIONS` block, so add it with the two `sentry_config_py_lines` above. Without it "Test Monitor" in the uptime UI returns an empty 400.

Upgrading
---------

1. Read the release notes of every version between the current and the target one: [hard stops](https://develop.sentry.dev/self-hosted/releases/) and the "Changes to Check" sections.
2. Rebuild `templates/docker-compose.yml` from the upstream `docker-compose.yml` of the target tag and put back the `deploy.resources.limits` blocks and `MAX_MEMORY_USAGE_RATIO` in the same places. A three-way merge with the previous tag as the base keeps them; a plain `patch` of the upstream diff fails when upstream changes an `image:` line right above a limits block.
3. Render the template and compare `docker compose config` of the rendered file with upstream: only the limits and the ratio may differ.
4. Bump `sentry_version` in `defaults/main.yml`. Consumers that pull the role from `main` must bump `sentry_version` in their inventory on the day the change is merged, otherwise the new template runs with old images.
5. Before the run, stop anything that restarts Sentry containers and take a backup: `pg_dump` of the `postgres` database and a disk snapshot. Recent releases ship migrations that drop tables and columns, so going back to the previous tag requires restoring the database.
6. Run the playbook limited to one host and watch `sentry_install_log-*.txt` in the project folder.

License
-------

MIT
