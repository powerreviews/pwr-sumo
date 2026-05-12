# pwr-sumo — RepoDocs
_Generated on 2026-05-11_

## Summary

### Overview
`pwr-sumo` is an Ansible role that installs and configures the Sumo Logic Collector agent on RedHat/Amazon Linux hosts so that local log files are shipped to PowerReviews' Sumo Logic tenant for centralized observability. It is a small infrastructure-provisioning building block: included by playbooks that bake AMIs or provision VMs across the org. Domain: log forwarding / observability agent provisioning.

### Tech Stack
| Category | Technology | Version |
|----------|-----------|---------|
| Language | YAML (Ansible) | _Not specified_ |
| Framework | Ansible role | `min_ansible_version: 1.2` (meta/main.yml) |
| Build Tool | _Not applicable (role consumed via Ansible Galaxy / playbook include)_ | _N/A_ |
| CI/CD | GitHub Actions | _Workflow at .github/workflows/secrets-scan.yml_ |
| Cloud/Infra | Sumo Logic (US2 region collector endpoint) | _RPM pulled from collectors.sumologic.com_ |

### Consumers
| Consumer | Type | How They Use It |
|----------|------|----------------|
| Ansible playbooks across PowerReviews infrastructure | Internal | Include the role to install the Sumo Logic Collector on hosts (`roles: - pwr-sumo`) — typical callers in this org would be `ansible-ami`, `ansible-playbooks`, `devops-playbooks`, but the role itself records no callers |
| Sumo Logic SaaS (US2) | External system | Receives logs forwarded by the collector this role installs (`https://collectors.us2.sumologic.com`) |
| GitHub Actions runner (ubuntu-latest) | External system | Runs the scheduled trufflehog secrets scan defined in `.github/workflows/secrets-scan.yml` |
| Slack (`#github-token-scan` channel) | External system | Receives failure notifications from the secrets-scan workflow via `rtCamp/action-slack-notify` |

### Dependencies on Org Repos
| Repo | Reason |
|------|--------|
| _None_ | `meta/main.yml` declares `dependencies: []`; the role contains no references to other org repos |

### External Integrations
| Service | Purpose | Integration Type |
|---------|---------|-----------------|
| Sumo Logic | Downloads collector RPM (`collectors.sumologic.com/rest/download/rpm/64`) and forwards logs to ingest endpoint (`collectors.us2.sumologic.com`) | REST (RPM download); agent uses Sumo Logic's protocol via collector daemon for log shipping |
| Slack | Notifies `github-token-scan` channel when trufflehog scan fails | Webhook (outbound) |
| TruffleHog (via `edplato/trufflehog-actions-scan`) | Scans git history for leaked secrets | SDK (GitHub Action) |

### Async & Scheduled Work
| Channel / Job | Type | Direction | Purpose |
|--------------|------|-----------|---------|
| `secret-scan` GitHub Actions workflow | Cron job (`0 14 * * 1-5` — 14:00 UTC Mon–Fri) | N/A (job) | Scans repo for committed secrets and alerts Slack on failure |
| Sumo Logic Collector daemon (`collector` service) | Background agent (managed by systemd; enabled via `systemctl enable collector`) | Produces | Tails configured log paths and ships records to Sumo Logic |

### Upgrade Alerts
| Dependency | Current Version | Issue | Severity |
|-----------|----------------|-------|----------|
| `actions/checkout@master` | Pinned to mutable `@master` | Unpinned action ref — supply-chain risk; `@master` was renamed to `@main` in the upstream action and v1 is long-deprecated (v4 is current) | Severe |
| `edplato/trufflehog-actions-scan@master` | Pinned to mutable `@master` | Unmaintained third-party action pinned to a mutable ref; community guidance has moved to `trufflesecurity/trufflehog` | Severe |
| `rtCamp/action-slack-notify@v2.0.2` | v2.0.2 (2020) | Five-plus years behind current; supersedes available | Severe |
| `min_ansible_version: 1.2` (meta) | Declares Ansible 1.2 support | Ansible 1.2 is long EOL; modules `get_url`, `yum`, `service`, `command`, `file`, `template` used here require modern Ansible — the declared floor is stale | Severe |

## API Reference

This repo is an **Ansible role**, not a service. Its public surface is its variables, tasks, handlers, and templates.

### Role Variables (consumer-tunable)

Declared in `defaults/main.yml` and documented in `README.md`:

| Variable | Default | Purpose |
|----------|---------|---------|
| `sumologic_config_dir` | `/opt/SumoCollector/config` | Directory for `user.properties` |
| `sumologic_config_dir_owner` | `root` | Owner of config dir / files |
| `sumologic_sources_dir` | `/opt/SumoCollector/sources` | Directory for collector source JSON |
| `sumologic_collector_url` | `https://collectors.us2.sumologic.com` | Sumo Logic ingest endpoint |
| `sumocollector_installer_rpm` | `https://collectors.sumologic.com/rest/download/rpm/64` | RPM download URL (RedHat only) |
| `sumologic_installer_rpm_local_folder` | `/tmp` | Local download path for RPM |
| `sumologic_collector_accessid` | `""` | Sumo Logic access ID (**required, secret**) |
| `sumologic_collector_accesskey` | `""` | Sumo Logic access key (**required, secret**) |
| `sumologic_collector_name` | `""` | Collector display name |
| `sumologic_collector_timezone` | `"UTC"` | Timezone applied to all source definitions |
| `sumologic_collector_log_paths` | `""` | List of `{name, path, use_multiline, category}` entries describing log sources |

### Tasks (`tasks/main.yml`)

Ordered task list executed when the role is invoked:

1. **Download SumoCollector redhat** — `get_url` of `sumocollector_installer_rpm` to `{{ sumologic_installer_rpm_local_folder }}/sumo_collector.rpm` (when `ansible_os_family == "RedHat"`).
2. **Install SumoCollector redhat** — `yum` install of the downloaded RPM; notifies `Restart SumoCollector`.
3. **Ensure sumologic_config_dir exists** — `file` (mode `0700`, owned by `sumologic_config_dir_owner`).
4. **Create sumo agent configuration file** — `template` renders `user.properties.j2` → `{{ sumologic_config_dir }}/user.properties`; notifies `Restart SumoCollector`.
5. **Ensure sumo sources directory exists** — `file` state=directory.
6. **Create collector sources configuration file** — `template` renders `collector.json.j2` → `{{ sumologic_sources_dir }}/sumologic-collector.json`; notifies `Restart SumoCollector`.
7. **Enable SumoCollector** — `command: systemctl enable collector` (when `ansible_service_mgr == "systemd"`).
8. **Start SumoCollector on boot** — `service` module with `enabled: yes` on the `collector` unit.

### Handlers (`handlers/main.yml`)

| Handler | Action |
|---------|--------|
| `Restart SumoCollector` | `service: name=collector state=restarted` (`changed_when: True`) |

### Templates

- **`templates/user.properties.j2`** — Renders the collector's `user.properties`: `url`, `accessid`, `accesskey`, `syncSources` path, `ephemeral=true`.
- **`templates/collector.json.j2`** — Renders a Sumo Logic Collector "sources" JSON with `api.version=v1` and a per-log-path `LocalFile` source entry (fields: `name`, `sourceType`, `automaticDateParsing`, `multilineProcessingEnabled`, `useAutolineMatching`, `forceTimeZone`, `timeZone`, `category`, `pathExpression`).

### Galaxy / Meta (`meta/main.yml`)

```
author: Mario Harvey
company: PowerReviews
description: Installs Sumo Logic
min_ansible_version: 1.2
dependencies: []
```

## Architecture

### System-context diagram

```
                                +-----------------------------+
                                |   Sumo Logic SaaS (US2)     |
                                | collectors.us2.sumologic.com|
                                +--------------^--------------+
                                               | logs (HTTPS, agent protocol)
                                               |
+-------------------+      +-----------------------------------------+
| Ansible control   |      |          Managed host (RHEL/EL)         |
| node /            |      |  +-----------------------------------+  |
| playbook caller   | -->  |  |   pwr-sumo role applied here      |  |
| (e.g. ansible-ami,|      |  |   - yum install sumo_collector.rpm|  |
|  ansible-playbooks|      |  |   - render user.properties        |  |
|  devops-playbooks)|      |  |   - render sumologic-collector.json|  |
+-------------------+      |  |   - systemd enable + start        |  |
                           |  +----------------+------------------+  |
                           |                   |                      |
                           |              tails log files             |
                           |                   v                      |
                           |       /var/log/... (caller-defined)      |
                           +-----------------------------------------+

  collectors.sumologic.com/rest/download/rpm/64  --(get_url at runtime)--> managed host

  GitHub repo  --(scheduled cron M-F 14:00 UTC)-->  trufflehog scan
                                                     | on failure
                                                     v
                                            Slack #github-token-scan
```

### Key components
- **`tasks/main.yml`** — Linear procedural play: download → install → write config → enable service → register on-boot.
- **`templates/user.properties.j2`** — The agent credentials + sync pointer. `ephemeral=true` means collectors do not persist registration if the host disappears (suited to autoscaled / immutable infra).
- **`templates/collector.json.j2`** — The list of `LocalFile` sources; the role expects callers to supply `sumologic_collector_log_paths` (otherwise the default `""` will fail to render as a list).
- **`handlers/main.yml`** — Single `Restart SumoCollector` handler triggered by RPM install or config-file change.
- **`defaults/main.yml`** — Centralizes URLs, paths, and required-but-blank credentials so callers must override secrets via group_vars / playbook vars.

### Data flow
1. Caller playbook invokes the role with credentials and a list of log paths.
2. Role downloads + installs the Sumo Collector RPM on RedHat-family hosts only.
3. Role renders `user.properties` (credentials + endpoint) and `sumologic-collector.json` (per-source local-file definitions).
4. `collector` systemd service is enabled and (re)started; agent then tails the configured paths and ships parsed events to `collectors.us2.sumologic.com`.

### CI/CD tooling
**GitHub Actions** — single workflow `.github/workflows/secrets-scan.yml`:
- **Trigger**: cron `0 14 * * 1-5` (weekdays 14:00 UTC).
- **Steps**: `actions/checkout@master` → `edplato/trufflehog-actions-scan@master` with `--regex --entropy=False --max_depth=1` → on failure, `rtCamp/action-slack-notify@v2.0.2` posts to `#github-token-scan` mentioning `@devops-team`.
- There is **no build/test/deploy pipeline** in this repo; the role is consumed in source form by downstream playbooks.

### Test architecture
_No tests present in the repo._

### Data model / database schema
_Not applicable — this role provisions an agent and writes config files; no databases are touched._

### Auth & trust boundaries
- **Inbound auth**: _Not applicable_ — the role runs locally on a managed host under Ansible's privilege escalation (`become: true`).
- **Outbound auth**:
  - Sumo Logic: `accessid` / `accesskey` rendered into `user.properties` (must be supplied by caller via group_vars / Vault).
  - Sumo Logic RPM download: anonymous HTTPS GET.
  - GitHub Actions → Slack: `${{ secrets.SLACK_WEBHOOK }}` webhook URL.
  - TruffleHog scan: `${{ secrets.ACCESS_TOKEN }}` GitHub token.
- **Authorization model**: _None at role level._ Filesystem hardening: config dir created with mode `0700` and owned by `sumologic_config_dir_owner` (defaults to `root`), keeping credentials root-readable only.

### Data ownership
_Not applicable — no datastores. The role manages on-host config files in `/opt/SumoCollector/{config,sources}` only._

### Deployment topology
_Deployment topology not in this repo._ The role itself targets RHEL-family hosts (`ansible_os_family == "RedHat"`) and assumes a systemd init (`ansible_service_mgr == "systemd"`), but the runtime, scaling model, and environments are determined by the consuming playbook.

## Repo Activity
Derived from git history; current HEAD is `3724f3d`.
- **Created**: 2017-06-07 (`c5ba8a1` — "first commit" by `badmadrad`).
- **Last meaningful change**: 2020-04-20 (`300aa23` — "forcing a systemd enable to fix an issue with links" by Daniel Enders), part of a same-day cluster of init-system fixes culminating in PR #1 merged 2020-04-21 (`e6dedb4`). The two later commits (`8b21453`, `3724f3d`, both 2020-07) only add/edit the secrets-scan workflow.
- **Activity level**: **0 commits in the last 90 days** (as of 2026-05-12). The repo has been effectively dormant since July 2020.
- **Hot spots** (top files by churn over the last 6 months): _No commits in the last 6 months — no hot spots._ For historical context, lifetime churn concentrates in `tasks/main.yml`, `.github/workflows/secrets-scan.yml`, and `templates/` (each touched 2–3 times).
- **Recent major changes**: _No major changes in the last 6 months._ Historically, the only notable changes are: (a) the 2017 initial implementation, (b) the April 2020 PR #1 ("tableau") that hardened service-manager handling on systemd, and (c) the July 2020 addition of the trufflehog-based secrets-scan GitHub Actions workflow.
