<!--
Copyright © Michal Čihař <michal@weblate.org>

SPDX-License-Identifier: CC0-1.0
-->

# Threat model: spoonnotfound/weblate

Status: built on the repository's own threat model, `docs/security/threat-model.rst` (linked from `SECURITY.md`). That document is the authority. This file summarizes it for triage and records the owner's answers (see "Owner decisions"). Facts marked *(draft)* are Devin's reading of the code.

## Owner decisions

- **Deployment:** a public instance with open registration. Any internet user can create an account, so a finding that requires only a regular authenticated account (no team or project role) is reachable by an anonymous attacker after self-registration. Requiring a login is not a mitigating precondition.
- **Critical assets:** all of them, namely host integrity (no command execution), user accounts / 2FA / API tokens, VCS credentials and repository integrity, private project and customer translation content, and availability.
- **Out of scope / accepted risk:** nothing beyond the upstream model. The upstream "Out of scope", "Security properties Weblate does not provide", "Known non-findings" and triage dispositions apply as written.
- **Infrastructure:** no infrastructure repository exists. Deployment controls (reverse proxy, body-size limits, network isolation of DB and datastore) are assumed as documented upstream.

## System overview

Weblate is a Django web localization platform: a WSGI/ASGI app behind a reverse proxy, with PostgreSQL, a Valkey/Redis datastore, Celery workers, a writable data directory holding the VCS checkouts, and optional outbound integrations (VCS hosts, IdPs, machine translation, CDN, backups). This repo is a fork of WeblateOrg/weblate. It holds no deployment manifests besides `dev-docker/` (development only). *(draft)*

## Components

| Component | Code | Trust side |
|---|---|---|
| **Web UI & REST API** | `weblate/trans/views/`, `weblate/api/`, `weblate/templates/`, `weblate/screenshots/`, `weblate/glossary/`, `weblate/memory/`, `weblate/workspaces/`, `weblate/legal/`, `weblate/metrics/` | Internet-facing |
| **Authentication & access control** | `weblate/accounts/` (login, 2FA, social auth pipeline, profile/data import), `weblate/auth/` (teams, permissions, scopes), `weblate/api/authentication.py`, `weblate/oidc/` (Weblate as OIDC provider) | Internet-facing, gate to everything |
| **VCS & webhooks** | `weblate/vcs/` (git, mercurial, GitHub App, SSH wrappers, GPG), `weblate/trans/views/hooks.py`, `weblate/trans/hooks/`, `weblate/gitexport/` (git HTTP export), `weblate/trans/repository*.py` | Public hooks in; child VCS processes and outbound SSH/HTTPS out |
| **File formats & backups** | `weblate/formats/`, `weblate/trans/backups.py`, `weblate/utils/zip.py`, `weblate/utils/backup.py` (Borg), translation upload in `weblate/trans/views/files.py` | Parses untrusted files and archives |
| **Add-ons & automation** | `weblate/addons/`, `weblate/automation/` (CEL helper), `weblate/utils/bulk_regex.py` / `regex_worker.py`, `weblate/trans/autotranslate.py` | Admin-configured execution |
| **Outbound integrations** | `weblate/machinery/`, `weblate/utils/outbound.py`, `weblate/addons/webhooks.py`, `weblate/addons/cdn.py`, `weblate/addons/fedora_messaging.py`, `weblate/accounts/avatar.py` | Outbound network (SSRF boundary) |
| **Background workers & admin** | `weblate/*/tasks.py`, `weblate/utils/celery.py`, `weblate/wladmin/`, `weblate/billing/`, `*/management/commands/` | Trusted same-instance execution |

## Trust boundaries and entry points

- **Client to Weblate:** browser and API requests from anonymous users, self-registered users and project-scoped API tokens become permission-checked actions (`weblate/auth/permissions.py`).
- **Webhook sender to Weblate:** public, unauthenticated forge hooks (`/hooks/...`) schedule repository updates when `ENABLE_HOOKS` and the project setting are on. Matching is by exact repository URL. Diagnostics (match counts, slugs) are documented disclosure. GitHub App webhooks are authenticated by URL token plus signature.
- **eTranslation callback:** public, accepted only with an unpredictable per-request token.
- **Weblate to local VCS:** repository URLs, branches, file masks and upstream content drive `git` and `hg` subprocesses (`weblate/vcs/base.py`, `git.py`, `ssh.py`).
- **Backup ZIP to filesystem:** uploaded project backups are restored subject to member, size and ratio limits.
- **Weblate to external services:** user-configurable outbound URLs reject private targets by default (`weblate/utils/outbound.py`). Operator allowlists and proxies widen that on purpose.
- **Weblate as OIDC provider:** relying parties registered by admins. Provider tokens do not grant REST API access.
- **Request to Celery worker:** queued VCS actions carry the initiating user, and the worker rechecks permission. Broker, datastore and workers are trusted.

## Assets (all critical per owner)

- Private project and restricted-component content: source strings, translations, screenshots, glossaries, translation memory.
- Repository integrity and the credentials Weblate holds: SSH keys, VCS tokens, GitHub App private keys, GPG signing key.
- User accounts: passwords, 2FA secrets, API tokens, sessions, e-mail addresses, profile data under the visibility policies.
- Host integrity: no command execution as the Weblate service user.
- Service availability within the documented limits.

## Attackers in scope (from the repo model)

- Unauthenticated internet client, including anyone who self-registers: public pages, registration, login, API, reachable hooks.
- Authenticated user / translator: acts within assigned teams and permissions.
- Reviewer / project manager / component admin: delegated project powers. Must not become site admin.
- Project-scoped API token holder.
- Webhook sender: forged, replayed or high-volume deliveries.
- External VCS or service provider: malicious repository content, API responses, redirects.
- Uploader of a malicious project backup.

## Out of scope / accepted risk (from the repo model; owner adds nothing)

- Compromised host, DB, datastore, proxy or worker. Malicious site admin or local operator. Deployment config (env vars, manifests) is trusted input.
- Third-party dependencies as such, build/release hygiene, third-party add-ons, tests, docs, dev fixtures, `dev-docker/`, examples.
- Actions within a documented permission: project managers changing VCS settings or credentials, `project.edit` reading restricted components or full backups, add-on managers operating on restricted descendants, `reports.view` covering its whole scope.
- Unauthenticated webhooks that only trigger modeled scheduling or documented diagnostics: hardening, not a vulnerability.
- Data published by explicit opt-in: shared TM contribution, CDN publication, public sharing of engage pages.
- Background work completing after the initiator's permissions change. Translation content dangerous to downstream renderers.
- Site-wide `group.edit` / `user.edit` granting broad power (documented misuse pattern).

## Security properties claimed (violations are valid findings)

Authorization separation across site, project, component, language, glossary, VCS, TM, screenshot, review and access scopes. Token scope limits. Session and 2FA integrity. No stored or reflected XSS in the Weblate origin. No VCS input becoming command execution. Private project data not disclosed to unauthorized users. Backup import not escaping its restore root. Outbound private-target restrictions enforced.

## Existing controls

Central permission checks (`weblate/auth/permissions.py`, `check_access`/`user.has_perm`). DRF auth and throttling (`weblate/api/authentication.py`, `throttling.py`, `ratelimits.py`). CSP and HTTPS settings. Backup import limits. Outbound URL validation (`weblate/utils/outbound.py`). Resource-limited regex and CEL helpers. Subprocess calls via argument lists in `weblate/vcs/`. Permission recheck in repository workers.

## Open questions

- None outstanding. Revisit if registration is closed, or if webhooks, machine translation or other integrations are disabled in your deployment.
