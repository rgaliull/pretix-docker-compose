# Pretix Docker deployment

This repository contains a public Docker Compose and Ansible deployment for
an operator-managed Pretix instance.

Automation agents should read [`AGENTS.md`](AGENTS.md) first. The detailed
upgrade and translation runbook is maintained below.

The application image is based on the official Pretix standalone image and
adds a private seating plugin. Production configuration, database
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

The target is defined in the ignored local file
`ansible/inventory/production.ini`, based on
`ansible/inventory/production.ini.example`. The playbook runs with sudo on
the host and deploys to the path configured in the local inventory.

Use the persistent local Python 3.13/Ansible environment for deployments:

```bash
source .venv/bin/activate
ansible --version  # Python 3.13, ansible-core 2.21+
```

Before the first run, create the ignored production inventory and verify that
the private plugin key path and private Git host are correct. The key is
consumed by Docker BuildKit via an SSH mount and is never copied into an image
layer. Do not add it, `.env`, `pretix.cfg`, or TLS files to Git.

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

## Runbook for future agents

This section is intentionally explicit so that a future operator or coding
agent can update the installation without relying on undocumented state.

### Scope and architecture

The production host is reached through the SSH target in the ignored local
inventory. The public repository is deployed to the configured deployment
directory; that directory is not a Git checkout. The Compose project is named
`pretix` and uses these containers:

| Container | Role |
| --- | --- |
| `pretix_app` | Pretix, Gunicorn, Celery, Nginx and cron |
| `pretix_db` | PostgreSQL |
| `redis` | Redis cache and broker |

The database and Pretix data are external Docker volumes discovered by the
playbook at runtime. Never replace them with newly named volumes during an
upgrade. The production-only files are kept on the host and deliberately
excluded from repository synchronization:

the deployment directory's `.env`, `pretix.cfg`, `nginx/nginx.conf`,
`crontab`, TLS files, certificate directories, and private plugin key.

The SSH key is used only through Docker BuildKit's SSH mount to install the
private seating plugin. It must never be copied into the build
context, Docker image, Ansible output, or Git.

### Required local environment

Use the Python 3.13 virtual environment prepared for this project:

```bash
source .venv/bin/activate
ansible --version
```

Create `ansible/inventory/production.ini` locally from
`ansible/inventory/production.ini.example` and fill in the operator's SSH
target, deployment directory, private plugin requirement, private Git host,
private key path, and public host. The real file is ignored by Git. Prefer an
Ansible vault for sensitive variables; never replace the example values in the
public file with production values.

Before making changes, verify the repository and SSH target without exposing
secrets:

```bash
git status --short
ansible -i ansible/inventory/production.ini pretix -m ping
```

All production actions must go through Ansible. Ad-hoc Ansible commands are
acceptable for read-only diagnostics. Do not use `docker compose` directly on
the production host for a deployment.

### Standard Pretix upgrade procedure

1. Inspect the current repository state, the pinned image tags, and the
   running service versions. Do not overwrite unrelated working-tree changes.
2. Review the Pretix release notes and compatibility requirements. A PostgreSQL
   major-version change is a separate migration project; changing the image
   tag alone is not a PostgreSQL upgrade procedure.
3. Update the pinned Pretix, PostgreSQL, or Redis versions in the public files
   only after checking compatibility. Keep application and database upgrades
   separate when possible.
4. Run the backup and validation before any build or restart:

   ```bash
   ANSIBLE_LOCAL_TEMP=/tmp/ansible-local \
     ansible-playbook -i ansible/inventory/production.ini ansible/deploy.yml \
     --tags backup
   ```

   The playbook discovers the active database volume and credentials from the
   running container, creates a compressed PostgreSQL custom-format dump under
   `/var/backups/pretix/`, and validates it with both `gzip` and `pg_restore`.
   Credentials are protected with Ansible `no_log` and are not printed.
5. Run the deployment:

   ```bash
   ANSIBLE_LOCAL_TEMP=/tmp/ansible-local \
     ansible-playbook -i ansible/inventory/production.ini ansible/deploy.yml
   ```

   This synchronizes only public files, recreates the image with BuildKit SSH
   forwarding, preserves external volumes, starts the stack, checks the
   Pretix runtime version, and checks HTTPS. A TLS check can fail during the
   first seconds of Nginx startup; if that happens, use a read-only Ansible
   health check with retries before deciding that the deployment failed.
6. Confirm the containers, runtime version, HTTPS endpoint, cron process, and
   database connectivity. Do not run migrations manually unless the Pretix
   release procedure explicitly requires them; the image's normal startup and
   the project playbook are the source of truth.
7. Record the release, image digest if available, backup filename, and any
   follow-up warnings in the change description or issue tracker.

### Updating or enabling languages

Pretix has three language categories: official, inofficial/community, and
incubating. An incubating language is not necessarily present in the stable
image's language registry. `allow_incubating` only makes a language selectable
after Pretix knows the language code; it does not create the language entry.

For Slovenian, the current repository therefore contains two public pieces:

* `docker/pretix/locale/sl/LC_MESSAGES/django.po` and `djangojs.po`, downloaded
  from Weblate's “original translation files” ZIP;
* `docker/pretix/patches/add-slovenian-language.patch`, which registers `sl` in
  Pretix's `ALL_LANGUAGES` and marks it incubating.

The Dockerfile compiles the PO files during `make production`. Do not commit
the downloaded ZIP or generated `.mo` files. The ZIP is ignored by Git and the
`.mo` files are generated inside the image.

To refresh Slovenian from Weblate:

1. Download the original translation-files ZIP from
   <https://translate.pretix.eu/download/pretix/-/sl/?format=zip>.
2. Inspect the archive with `unzip -l`; do not extract production files or
   credentials from it.
3. Replace only the core files in
   `docker/pretix/locale/sl/LC_MESSAGES/`:

   ```text
   pretix/pretix/src/pretix/locale/sl/LC_MESSAGES/django.po
   pretix/pretix/src/pretix/locale/sl/LC_MESSAGES/djangojs.po
   ```

4. Validate the PO files locally with `msgfmt --check`; warnings about the
   initial `Project-Id-Version` header are not secrets and are usually benign.
5. Check whether the current Pretix base image already contains `sl` in
   `pretix/_base_settings.py`. If it does, do not add or refresh the source
   patch unnecessarily. If it does not, retain/update the minimal patch and
   verify its context against the new Pretix source.
6. Keep the production configuration as follows unless the project owner
   explicitly requests another default:

   ```ini
   [locale]
   default=en

   [languages]
   allow_incubating=sl
   ```

   Do not add an `enabled` option when the goal is the widest possible list of
   languages. An `enabled=` value is an allow-list and restricts the normal
   Pretix languages.
7. Apply the production language configuration only through Ansible:

   ```bash
   ANSIBLE_LOCAL_TEMP=/tmp/ansible-local \
     ansible-playbook -i ansible/inventory/production.ini \
     ansible/enable-slovenian.yml
   ```

   This keeps `default=en`, enables `allow_incubating=sl`, removes a stale
   restrictive `enabled=` option, preserves the config's protected ownership,
   restarts Pretix, and verifies the language code.
8. After deployment, test both the organizer interface and a real event
   checkout. Slovenian is incomplete; untranslated strings are expected to
   fall back to English. Adding `sl` to the global list does not automatically
   make it available for every event: an organizer may still need to select it
   in the event's localization settings.

### Configuration permissions

The mounted `pretix.cfg` contains production settings and must not be public.
The container runs application workers as UID/GID `15371` (`pretixuser`). The
Ansible language playbook keeps the file as `root:15371` with mode `0640`, so
the application can read it without making it world-readable. If the base
image changes the `pretixuser` UID/GID, update the playbook after verifying the
new value inside the image.

### Troubleshooting checklist

* **Language is absent:** verify the PO files, compiled MO files, the
  `ALL_LANGUAGES` registration, `allow_incubating=sl`, and that the running
  container was rebuilt rather than merely restarted.
* **Language check shows defaults:** ensure diagnostics set
  `PRETIX_CONFIG_FILE=/etc/pretix/pretix.cfg` and that the mounted config is
  readable by `pretixuser`.
* **Cron warning:** inspect the running cron process and `/tmp/crontab` through
  Ansible. The configured periodic job runs at the schedule in the tracked
  `docker/pretix/crontab`; a warning immediately after restart can be
  transient until the next scheduled run.
* **HTTPS check fails immediately after restart:** inspect `docker compose ps`
  and retry HTTPS after Nginx has finished starting. Do not rebuild or touch
  the database solely because of an early TLS check.
* **Build tries to expose a key:** stop immediately. The key must be supplied
  only with `docker compose build --ssh`; remove any key from the context and
  rotate it if it was exposed.

### Git and secret-safety rules

Before committing, run `git status --short`, `git diff --check`, and inspect
the staged file list. Never commit `.env`, `pretix.cfg`, TLS private keys,
database dumps, private plugin keys, Weblate ZIP archives, or generated
production artifacts. Use a descriptive commit and push only after a human
reviews the staged diff.

## Local validation

Without production secrets, validate the public Compose model with a temporary
`.env` containing non-production values and run:

```bash
docker compose config
```

Never commit that `.env` file.

## License

This product is available under the Apache 2.0 license.
