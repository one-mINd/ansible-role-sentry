Ansible Role: peaceman.sentry_selfhosted
========================================

Installs sentry selfhosted via docker-compose project from

https://github.com/getsentry/onpremise

Requirements
------------

* Installed docker and docker-compose

Role Variables
--------------

```yaml
sentry_docker_compose_project_folder: /opt/docker/sentry
sentry_version: 26.8.0
sentry_bind: 9000
sentry_env: {}

sentry_mail:
  host: mail.example.com
  from: sentry@example.com
  port: 587
  username: redacted
  password: redacted

sentry_config: {}
sentry_config_py: {}

sentry_url: https://sentry.example.com
sentry_superusers:
  - email: admin@example.com
    password: redacted

sentry_apply_automatic_config_updates: "0"

```
