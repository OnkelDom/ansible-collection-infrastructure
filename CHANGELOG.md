# Changelog

## 2.2.0 - 2026-11-03

- Add `logging` for configurable journald limits and optional delegation of global logrotate policy.
- Add opt-in, staged and backed-up main configuration management to `logrotate`; existing rule-only usage stays unchanged.
- Add container configuration checks and guarded dedicated-VM persistence checks.

## 2.1.0 - 2026-10-29

- Add `kernel_crash_recovery` with persistent panic-on-Oops, a configurable reboot delay, live drift correction and explicit rollback.
- Add a serial rollout playbook and container-safe configuration tests; runtime activation is rejected in containers.

## 2.0.1 - 2026-10-27

- Make Debian-family Telegraf repository key conversion idempotent while preserving key rotation and missing/empty keyring recovery.
- Exercise unchanged runs, rotation, recovery and repository opt-out with real Ansible modules, HTTP downloads and GPG conversion.

## 2.0.0 - 2026-10-27

### Shared repository standard

- Align README, contribution/security docs, Make wrappers and local/CI entry points with degitag-collections-v1.
- Verify repository contracts, collection metadata, archive boundaries and payload hashes with shared tools.
- Preserve role APIs, platform licenses and historical validation evidence; runtime acceptance remains separate.

### Collection namespace

- Rename the host collection from `degit.infrastructure` to `degitag.infrastructure`, including role/filter references, generated examples, CI artifacts and validation tooling.
- Keep host ownership markers and the namespaces of separate collections unchanged; dependent consumers require an explicit reference update.

### UKHD function transfer

- Add shared proxy credential normalization, system pip proxy configuration and protected Docker daemon proxy support with immediate handler application.
- Add opt-in maintenance reboots with a portable, testable requirement controller and a generic interactive Bash prompt role.
- Extend CA verification, selective audit logging controls, SSH-only SSSD groups, explicit user groups and NGINX stream proxies.
- Prefer AEAD SSH ciphers with AES-CTR compatibility and an AES-only EL9 platform list; preserve custom-port service behavior.
- Keep PRTG, Postfix, session logging and customer-specific GitLab proxy files outside this transfer.

### Collection architecture

- Replace relative test-role references with real collection roles and verify CI output formatting.
- Isolate SFTP authorized keys per account and correct Debian 13 Dante packages and native configuration rendering.
- Move the public collection identity to degitag.infrastructure and extract seven K3s roles into degitag.k3s.
- Replace six public firewall/network backends with fact-selected firewall and network roles.
- Add exact release validation and layered family/distribution/release variables to every role.
- Upgrade the controller baseline to Ansible Core 2.20/2.21 and current dependency majors.
- Add typed portable APIs, strict CI, sanity checks, artifact inspection, five-platform smoke tests and tag releases.
- Correct service-handler switches, package overrides, backup permissions/defaults and EL9 auditd/EPEL behavior.
- Make static resolver ownership and scheduled Borg backups explicit opt-ins; require real SMTP and directory inputs.
- Update Node Exporter, Traefik, borgmatic and the pinned uv bootstrap; make MOTD generation idempotent.

Alle Änderungen an der `lenhardt_its.base` Ansible Collection werden in diesem Dokument festgehalten.

## [1.0.0] - 2025-03-05
### Hinzugefügt
- Erste Version der **`lenhardt_its.base`** Ansible Collection.
- Basisrollen für die Standardserverkonfiguration.

### Geändert
- `ansible-lint` Korrekturen: Über 100 Fehler und Warnungen behoben.
  - Veraltete Module ersetzt.
  - Loop-Fehler (`loop` + `with_items`) bereinigt.
  - Idempotenz sichergestellt (`changed_when`, `when`-Bedingungen angepasst).
  - Variablenbenennung (`var-naming`) standardisiert.

### Behoben
- Fehlerhafte `loop`-Strukturen durch `subelements()` ersetzt.
- Syntax-Fehler in `tasks/main.yml` korrigiert.
- Nicht unterstützte Module entfernt oder angepasst.

---

## Nächste Schritte
- Testsuite mit **Molecule** und **GitHub Actions** erweitern.
- Dokumentation für alle Rollen verbessern.
- Automatisierte CI/CD-Checks für Codequalität hinzufügen.