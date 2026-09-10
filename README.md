# Pretix for Upsala Circus

This repository contains the public Docker Compose and Ansible deployment for
the production Pretix instance at <https://tickets.upsalacircus.de/>.

The application image is based on the official Pretix standalone image and
adds the private `pretix-seating` plugin. Production configuration, database
credentials, TLS material, and the SSH key for the private plugin are kept on
the server or in a local Ansible vault. They are intentionally not stored in
this public repository.

## Versions

The versions are pinned in `docker-compose.yml` and
`docker/pretix/Dockerfile`:

| Component | Version |
| --- | --- |
| Pretix image | `pretix/standalone:2026.7.0` |
| Pretix runtime | `5.2.16` |
| PostgreSQL | `16.15-alpine3.24` |
| Redis | `8.10.1-alpine3.23` |

Pretix is updated to the latest stable release known when the change is made.
PostgreSQL major upgrades are not performed by changing the image tag; they
require a separate, tested `pg_upgrade` or logical-migration procedure.

## Slovenian interface

The Slovenian translation is an incubating Pretix translation. The public PO
files downloaded from [Weblate](https://translate.pretix.eu/projects/pretix/-/sl/)
are included in the image build under `docker/pretix/locale/` and compiled by
Pretix's production build. It is enabled for user selection with
`allow_incubating=sl`, while the default remains `default=en`.

The translation is incomplete, so untranslated strings may still appear in
English. Refresh the PO files from Weblate when updating the translation.

## Production deployment with Ansible

The target is defined in `ansible/inventory/production.ini` and uses the SSH
alias `tickets` from the operator's SSH config. The playbook runs with sudo on
the host and deploys to `/opt/pretix`.

Use the persistent local Python 3.13/Ansible environment for deployments:

```bash
source /Users/ramil/.venvs/ansible313/bin/activate
ansible --version  # Python 3.13, ansible-core 2.21+
```

Before the first run, verify that `/opt/pretix/id_rsa` on the server is the
read-only deploy key for `code.rami.io`. It is consumed by Docker BuildKit via
an SSH mount and is never copied into an image layer. Do not add it, `.env`,
`pretix.cfg`, or TLS files to Git.

Run the database backup and verify it without changing the application:

```bash
ANSIBLE_LOCAL_TEMP=/tmp/ansible-local \
  ansible-playbook -i ansible/inventory/production.ini ansible/deploy.yml \
  --tags backup
```

Deploy only after the backup task succeeds:

```bash
ANSIBLE_LOCAL_TEMP=/tmp/ansible-local \
  ansible-playbook -i ansible/inventory/production.ini ansible/deploy.yml
```

The playbook preserves the existing external Docker volumes, validates the
Compose configuration, builds the app with the private plugin, recreates the
services, verifies the Pretix version, and performs an HTTPS health check.
Backups are stored on the server under `/var/backups/pretix/`.

## Local validation

Without production secrets, validate the public Compose model with a temporary
`.env` containing non-production values and run:

```bash
docker compose config
```

Never commit that `.env` file.

## License

This product is available under the Apache 2.0 license.
