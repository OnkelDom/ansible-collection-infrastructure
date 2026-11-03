# logging

Configure a portable journald and logrotate host logging baseline.

## Supported platforms

- Ubuntu 22.04, 24.04, 26.04
- Debian 13
- Rocky Linux 9 / RHEL 9

## Architecture and operation

Facts are collected at entry only when the required platform facts are missing.
`tasks/preflight.yml` rejects unsupported releases before making changes and loads
family, distribution and release variables from `vars/platform/` in that order.
Public inputs belong in `defaults/main.yml`; platform internals belong in `vars/`.

### Policy and ownership

This role owns `/etc/systemd/journald.conf.d/60-degitag-logging.conf` and delegates
text log rotation to `degitag.infrastructure.logrotate`. journald rotates its own
binary journals. rsyslog, journal upload and journal namespaces are separate responsibilities.
No external repositories are needed.

The defaults select persistent storage, `SystemMaxUse=256M`, `SystemKeepFree=1G`
and `RuntimeMaxUse=128M`. These are journal storage limits and a free-space target,
not filesystem quotas or a guaranteed retention period. Size inputs accept integer
bytes or a K/M/G/T/P/E suffix. Optional empty inputs inherit the systemd default.
`logging_journald_max_retention_sec` accepts durations such as `7d` or `1h 30min`.

APT metadata is refreshed before managed package installation, with a default
validity of 3600 seconds. Set `logging_manage_package_cache=false` when cache
maintenance belongs to another baseline component. `logging_package_cache_valid_time`
accepts a nonnegative number of seconds. Metadata refresh is excluded from
configuration idempotence counts, consistently with the collection package role.

### Activation and migration

`logging_journald_manage_package` controls native package installation. An empty
`logging_journald_packages` list selects the platform package; a nonempty list replaces it.
`logging_journald_manage_service=false` renders the policy without restarting or
flushing journald. Live management requires systemd. Handlers restart an already
running service and flush persistent storage; a stopped service stays stopped.
The static journald unit is not enabled or disabled. No forced rotation or vacuum runs.

An existing `limits.conf` from the quickfix is rejected before configuration changes.
Inspect it with `vim`, migrate its settings into inventory and remove it explicitly.
Other operator fragments are preserved. Review `systemd-analyze cat-config
systemd/journald.conf` for conflicting overrides. This command displays the composed
configuration; it is not a syntax validator. The role validates its supported input
values before writing. Live parser diagnostics require VM acceptance.

`logging_manage_logrotate` controls delegation. Set
`logging_manage_logrotate_global=true` only after reviewing ownership of the main
logrotate file and any existing top-level rules. Its default is false. Configure
rotation values and custom text-log rules through the public `logrotate_*` inputs.
See the logrotate README for staged validation, backups and rollback.

Removing the role or setting a management flag to false leaves existing configuration
in place. To restore the underlying journald policy, explicitly remove the owned
fragment and restart the active journal. Old journal files are not deleted.

### Acceptance

Use `bash tests/run_role_functional_checks.sh logging debian:13` for disposable
container configuration tests. They cover overrides, invalid inputs, opt-out,
check mode and staged validation, with service management disabled. Fixtures are
created and removed only in disposable test guests; their setup is excluded from
the role idempotence count.

Use `tests/integration/logging.yml` against the `logging_test_vms` inventory group
with `logging_runtime_test=true` for real journal persistence and SELinux checks.
Run twice and require `changed=0` on the second run. Keep platform results and
remaining limitations in `docs/validation/`; Rocky does not replace a real RHEL test.


## Role Variables

The role interface is validated through `meta/argument_specs.yml`. Defaults are defined in `defaults/main.yml`.

```yaml
---
logging_manage_journald: true
logging_journald_storage: persistent
logging_journald_system_max_use: 256M
logging_journald_system_keep_free: 1G
logging_journald_runtime_max_use: 128M
logging_journald_runtime_keep_free: ''
logging_journald_system_max_file_size: ''
logging_journald_runtime_max_file_size: ''
logging_journald_max_retention_sec: ''
logging_journald_manage_package: true
logging_journald_packages: []
logging_journald_manage_service: true
logging_manage_logrotate: true
logging_manage_logrotate_global: false
logging_manage_package_cache: true
logging_package_cache_valid_time: 3600
```

## Example Playbook

```yaml
- name: Configure host logging baseline
  hosts: linux_hosts
  become: true
  gather_facts: true
  roles:
    - role: degitag.infrastructure.logging
      vars:
        logging_manage_logrotate_global: true
```

## Testing

Build and install the current collection checkout before running syntax checks, so the
fully qualified role reference resolves to this implementation.

Run `python tools/generate_role_scaffold.py --check`, `pytest tests/unit` and
`bash tests/run_role_syntax_checks.sh` from the collection root.
CI additionally runs YAML lint, Ansible lint, collection build/install and container smoke tests.
VM functional results are tracked separately; metadata lists intended compatibility,
not proof that every feature has passed on every distribution.

## Dependencies

Install the collection dependencies from `requirements.yml`. Role dependencies:
`[]`.

## Change management

Use check mode where the underlying modules support it. Review the diff before applying
changes. Network, firewall, identity and storage changes need an out-of-band recovery
path. Secrets belong in Ansible Vault or the operator secret store, never in defaults.
