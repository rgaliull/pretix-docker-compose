# Agent instructions

## Scope

This repository contains a public Docker Compose definition and Ansible
playbooks for an operator-managed Pretix installation. Keep this repository
free of production identifiers, credentials, private keys, TLS material,
database dumps, and private Git URLs.

Read `README.md` before changing deployment or translation behavior. It is the
detailed operational runbook; this file is the short safety and workflow
contract for automated agents.

## Required workflow

- Inspect `git status --short` before editing and preserve unrelated changes.
- Use the Python 3.13 Ansible environment configured by the operator.
- Use Ansible for all production actions. Direct Docker commands are limited to
  read-only diagnostics performed through Ansible.
- Create and validate a PostgreSQL backup before any production build or
  restart. Use `ansible/deploy.yml --tags backup`.
- Do not build, test against production, restart services, or deploy unless the
  user explicitly asks for that action in the current request.
- Do not push commits unless the user explicitly asks for a push.
- After edits, run `git diff --check` and scan the staged content for secrets
  and production-specific identifiers.

## Public repository safety

- `ansible/inventory/production.ini` and `docker/pretix/pretix.cfg` are local
  ignored files. Use their public `.example` templates as documentation.
- Private plugin settings are supplied through ignored inventory variables and
  Docker Compose build arguments. The private plugin key must be exposed only
  through a BuildKit SSH mount.
- Never hard-code a production hostname, organization name, operator username,
  private Git hostname, SSH key path, password, token, or real domain in a
  tracked file.
- Do not add `.env`, `*.env`, TLS keys/certificates, backups, Weblate ZIP files,
  or generated `.mo` files.

## Pretix and translations

- Keep `default=en` unless the user explicitly requests a different default.
- Keep the widest language list by omitting `languages.enabled`; it is a
  restrictive allow-list.
- Slovenian is incubating in this deployment and is enabled with
  `allow_incubating=sl`.
- The public Slovenian sources are tracked at
  `docker/pretix/locale/sl/LC_MESSAGES/django.po` and `djangojs.po`.
- If the base image does not register `sl`, retain the minimal patch at
  `docker/pretix/patches/add-slovenian-language.patch`.
- Refresh translation sources from Weblate's original translation-files ZIP,
  validate them with `msgfmt --check`, and never commit the ZIP or compiled
  output.

## Change handoff

Describe what changed, what was deliberately not run, backup status, and any
remaining deployment steps. Keep production deployment separate from ordinary
repository edits unless the user explicitly combines them.
