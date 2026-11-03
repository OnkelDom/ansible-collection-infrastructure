# logrotate

Manage custom logrotate rules for application and service logs.

## Supported platforms

- Ubuntu 22.04, 24.04, 26.04
- Debian 13
- Rocky Linux 9 / RHEL 9

## Architecture and operation

Facts are collected at entry only when the required platform facts are missing.
`tasks/preflight.yml` rejects unsupported releases before making changes and loads
family, distribution and release variables from `vars/platform/` in that order.
Public inputs belong in `defaults/main.yml`; platform internals belong in `vars/`.

### Global policy adoption

Custom rules retain their existing API. `logrotate_manage_global_config=false`
preserves the distribution main file. Explicit true takes ownership of
`logrotate_global_config_file` (default `/etc/logrotate.conf`) and replaces its
contents with backup enabled. Review and migrate required top-level rules to
`logrotate_rules` or operator-owned include files before adoption. Package fragments
in the include directory are preserved.

Global defaults are daily rotation, seven archives, file creation, date suffixes
and compression. They precede the include directory; individual rules can override
them. Disabling any boolean emits its inverse directive explicitly.
The global `su` context defaults to `root root` and is configurable through
`logrotate_global_su_user` and `logrotate_global_su_group`. This explicit context
also supports group-writable Ubuntu log directories without assuming that a
syslog group exists on minimal installations. Both identities must exist on the
target; native validation rejects unknown accounts before replacing the main file.

### Validation and scheduling

The native `logrotate -d -s /dev/null` validates the staged main file, including
installed fragments, before replacement. It rotates no files and writes no state.
A validation error leaves the old main file intact. The include path must be absolute
and contain only letters, digits, underscores, dots, slashes or hyphens.

A distribution timer or cron entry must exist. The role neither creates a second
scheduler nor enables an inactive scheduler. Verify actual scheduling with
`systemctl list-timers` and the native cron service on the target host. On a fresh
host, check mode cannot validate using a not-yet-installed binary; install the
package first.

### Rollback

Disable global management and restore the explicitly selected Ansible backup through
a controlled copy; use `vim` to inspect differences first. Disabling management alone
leaves the current file in place. Global ownership never authorizes removal of
unrelated package rules.


## Role Variables

The role interface is validated through `meta/argument_specs.yml`. Defaults are defined in `defaults/main.yml`.

```yaml
---
logrotate_manage_package: true
logrotate_package_state: present
logrotate_config_dir: /etc/logrotate.d
logrotate_file_prefix: degit-
logrotate_prune_managed_files: false
logrotate_rule_defaults:
  state: present
  frequency: weekly
  rotate: 8
  missingok: true
  notifempty: true
  compress: true
  delaycompress: false
  copytruncate: false
  sharedscripts: false
  dateext: false
  dateyesterday: false
  create: false
  create_mode: '0640'
  create_owner: root
  create_group: root
  su_user:
  su_group:
  olddir:
  extension:
  dateformat:
  size:
  minsize:
  maxsize:
  minage:
  maxage:
  start:
  mail:
  mailfirst: false
  maillast: false
  firstaction: ''
  prerotate: ''
  postrotate: ''
  lastaction: ''
  extra_directives: []
logrotate_rules: []
logrotate_manage_global_config: false
logrotate_global_config_file: /etc/logrotate.conf
logrotate_global_frequency: daily
logrotate_global_rotate: 7
logrotate_global_create: true
logrotate_global_dateext: true
logrotate_global_compress: true
logrotate_global_su_user: root
logrotate_global_su_group: root
```

## Example Playbook

```yaml
- name: Configure custom log rotation policies
  hosts: linux_hosts
  become: true
  gather_facts: true
  roles:
    - role: degitag.infrastructure.logrotate
      vars:
        logrotate_rules:
          - name: myapp
            paths:
              - /var/log/myapp/*.log
            frequency: daily
            rotate: 14
            compress: true
            delaycompress: true
            create: true
            create_mode: "0640"
            create_owner: myapp
            create_group: myapp
            sharedscripts: true
            postrotate: |
              systemctl reload myapp.service >/dev/null 2>&1 || true
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
