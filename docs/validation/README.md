# Validierung der Collection 2.0.0

Neue Rolle und Testrollout vom 29.10.2026: [kernel-crash-recovery-20261029.md](kernel-crash-recovery-20261029.md).

Repo-uebergreifender Abschluss: [repository-family-2026-10-27.md](repository-family-2026-10-27.md).

Aktueller Struktur-Nachweis: [repository-alignment-20261027.md](repository-alignment-20261027.md).

Namespace-Pruefung vom 2026-10-27: siehe [namespace-degitag.md](namespace-degitag.md). Historische Nachweise und Hashes bleiben unveraendert.

Stand der urspruenglichen Funktionsabnahme: 2026-10-18.

`results.json` dokumentiert die urspruengliche Refaktorierungs-Abnahme vor der Umbenennung der Host-Collection. Aktuelle Pruefungen der Namensaenderung werden separat in [rename-infrastructure.json](rename-infrastructure.json) festgehalten; historische Artefakt-Hashes und VM-Ergebnisse werden nicht nachtraeglich umetikettiert.
 Lokale Pruefung des vorbereiteten Zustands; GitHub Actions laufen erst nach Uebertragung und Push.

## Testmatrix

| System | Testbasis | VM-Funktionstest | Container-Smoke |
| --- | --- | --- | --- |
| Debian 13 | offizielles Generic Cloud Image | bestanden | siehe Ergebnisdatei |
| Ubuntu 22.04 | offizielles Cloud Image | bestanden | siehe Ergebnisdatei |
| Ubuntu 24.04 | offizielles Cloud Image | bestanden | siehe Ergebnisdatei |
| Ubuntu 26.04 | offizielles Cloud Image | bestanden | siehe Ergebnisdatei |
| Rocky Linux 9.8 | offizielles Generic Cloud Image, SELinux enforcing | bestanden | siehe Ergebnisdatei |
| RHEL 9 | keine Subscription-Testmaschine verwendet | nicht ausgefuehrt | nicht ausgefuehrt |

## Nachgewiesenes Verhalten

- Plattform-Preflight aller auf dem jeweiligen System zugelassenen Rollen: Familie, Distribution, Release, Facts und Ablehnung unpassender Plattformen. Wiederholung ohne Aenderung.
- `network`: zusaetzliche isolierte NIC mit statischem IPv4/IPv6 ueber Netplan, ifupdown bzw. NetworkManager; persistente Adressen nach echtem VM-Neustart vor erneutem Ansible-Abgleich; Wiederholung `changed=0`.
- `firewall`: aktive UFW-/firewalld-Regeln, HTTP-Verbindungen ueber IPv4 und IPv6 auf freigegebenen Ports und verweigerte Verbindungen auf einem nicht freigegebenen Port. SSH blieb erreichbar.
- Basisdienste: Pakete, CA, Logrotate, Chrony, QEMU Guest Agent, Auditd, Fail2ban, SSH und BorgBackup. Native Validatoren fuer SSH und Fail2ban sowie laufende Dienste; zweiter Lauf auf allen fuenf VMs `changed=0`.
- Agenten: Node Exporter 1.12.1 mit HTTP-Metrikabruf, Traefik 3.7.15 mit HTTP-Antwort und Borgmatic 2.1.10 mit realem Versionsaufruf; zweiter Lauf auf allen fuenf VMs `changed=0`.
- SFTP: echte Schluesselauthentifizierung, Upload, Download mit Inhaltsvergleich und Loeschen einer Testdatei im Chroot auf allen fuenf VMs. Ein zweites Konto konnte sich mit dem Schluessel des ersten Kontos auf keinem der fuenf Systeme anmelden. Benutzerschluessel liegen getrennt unter `/etc/ssh/authorized_keys/`; SELinux blieb auf Rocky aktiv.
- Docker wurde ueber die Collection auf der Ubuntu-24.04-Test-VM installiert. Container-Smoke und Dante-/WPAD-Konfigurationstests laufen mit dem Controller ausserhalb des Containers, jeweils zweifach mit gefordertem `changed=0` im zweiten Lauf.

## Reproduzierbare Checks

```bash
.venv/bin/python tools/generate_role_scaffold.py --check
.venv/bin/ansible-galaxy collection build --force
.venv/bin/ansible-galaxy collection install degitag-infrastructure-2.0.0.tar.gz --force --no-deps
.venv/bin/python -m pytest tests/unit
.venv/bin/yamllint .
ANSIBLE_HOME="$PWD/.ansible" .venv/bin/ansible-lint
PATH="$PWD/.venv/bin:$PATH" bash tests/run_role_syntax_checks.sh
.venv/bin/ansible-galaxy collection build --force
.venv/bin/python tools/check_collection_artifact.py degitag-infrastructure-2.0.0.tar.gz
PATH="$PWD/.venv/bin:$PATH" bash tests/run_collection_sanity.sh degitag-infrastructure-2.0.0.tar.gz infrastructure
```

Die Artefaktpruefung kontrolliert Namespace, Version und Ausschluss lokaler Toolchains, Testdaten und Caches. Sanity fuehrt `compile` und `import` ausserhalb des Git-Checkouts aus. Der Test ist kein vollstaendiger `ansible-test sanity`-Lauf aller verfuegbaren Pruefer.

Container: `SMOKE_CONTAINER_IMAGE=debian:13 bash tests/run_container_integration_smoke.sh`; Funktionstests: `bash tests/run_role_functional_checks.sh dante debian:13` bzw. `wpad`. Die CI wiederholt diese Checks fuer die fuenf Distributionen. VM-Playbooks stehen in `tests/integration/platform.yml`, `services.yml`, `agents.yml`, `sftp.yml`, `firewall_traffic.yml` und `reboot.yml`; ausschliesslich mit einem dedizierten Test-Inventar ausfuehren, da Netzwerk, Firewall und Neustarts real angewendet werden.

## Grenzen

Keine Freigabe beliebiger Produktionskonfigurationen aus einem erfolgreichen Baseline-Test ableiten. Insbesondere AD-/SSSD-Join, ACME-Ausstellung, echte Backup-Restore-Ketten, Storage-Rekonfigurationen, jede VLAN-/Bond-/Bridge-Topologie und alle optionalen Anwendungsintegrationen wurden nicht vollstaendig durchgespielt. Netzwerk-Compiler und Eingabegrenzen haben Unit-Tests; das ersetzt keinen Topologietest mit dem eigenen Switch und IP-Plan. RHEL nutzt denselben EL9-Pfad wie Rocky, braucht aber eine gesonderte Subscription-/Repository-Abnahme.

Ergebniszusammenfassungen ohne Zugangsdaten liegen in [results.json](results.json). Die fuenf dafuer erstellten Proxmox-Gaeste, Seed-ISOs und heruntergeladenen Cloud-Images wurden nach Abschluss entfernt. Vorhandene Gaeste und produktive Cluster blieben unveraendert.

## Paketbedingte Ausnahme

Die Host-Rolle `dante` unterstuetzt Ubuntu und Rocky/RHEL 9, nicht Debian 13. Der echte Debian-13-Test hat die fehlenden Stable-Pakete nachgewiesen; der Preflight lehnt dieses Ziel vor Aenderungen ab. [Debian-Paketsuche](https://packages.debian.org/search?keywords=dante-server) und [Debian-Projektdiskussion](https://lists.debian.org/debian-project/2025/09/msg00000.html). Die CI-Funktionsmatrix umfasst daher vier Dante- und fuenf WPAD-Ziele.

## Logging-Baseline

[Validierung vom 3. November 2026](logging-20261103.md): neue Logging-Rolle, globale Logrotate-Policy, Container-Matrix und dedizierte VM-Laufzeitpruefungen inklusive Dienstschutz und SELinux.
