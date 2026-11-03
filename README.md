# Ansible Collection `degitag.infrastructure`

[![CI](https://github.com/degitag/ansible-collection-infrastructure/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/degitag/ansible-collection-infrastructure/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/degitag/ansible-collection-infrastructure)](https://github.com/degitag/ansible-collection-infrastructure/releases)
[![Tag](https://img.shields.io/github/v/tag/degitag/ansible-collection-infrastructure?sort=date)](https://github.com/degitag/ansible-collection-infrastructure/tags)
![Ansible Core](https://img.shields.io/badge/ansible--core-2.20%20%7C%202.21-blue)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Reproduzierbare Betriebssystem-Baseline und allgemeine Linux-Dienste. Host-Netzwerk und Firewall haben jeweils eine gemeinsame API; Kubernetes- und K3s-spezifische Aufgaben liegen in [`degitag.k3s`](https://github.com/degitag/ansible-collection-k3s).

## Verantwortung und Architektur

- `firewall`: Debian/Ubuntu → UFW, Rocky/RHEL → firewalld. Dienstrollen reichen ihre eigenen Portregeln an diese Rolle weiter. Regeln sind additiv; keine impliziten Resets fremder Regeln.
- `network`: Ubuntu → Netplan/networkd, Debian → ifupdown, Rocky/RHEL → NetworkManager/nmcli. Eine gemeinsame Interface-Struktur beschreibt IPv4/IPv6, DHCP, DNS, Routen, VLANs, Bridges und Bonds.
- Jede Rolle beginnt mit einem generierten Preflight. Fehlende benoetigte Facts werden nachgeladen; vorhandene Facts werden wiederverwendet. Nicht unterstuetzte OS-Versionen werden vor Aenderungen abgewiesen.
- Plattformwerte werden in `vars/platform/` von Familie ueber Distribution bis zur Release-Datei geschichtet. Leere Dateien bedeuten geerbte Werte; kein stiller Fallback auf fremde Betriebssysteme.
- Defaults beschreiben die oeffentliche API, `meta/argument_specs.yml` typisiert sie. Interne Paket-, Pfad-, Service- und Validatordaten liegen in `vars/`.

### Operator-Erweiterungen

`proxy` normalisiert Anmeldedaten fuer Systemwerkzeuge und pip; `docker` uebernimmt denselben Proxy-Vertrag fuer den Daemon. Die Rollen [`conditional_reboot`](roles/conditional_reboot/README.md) und [`prompt`](roles/prompt/README.md) stellen allgemeine Wartungs- und Shell-Funktionen bereit. Wartungsneustarts bleiben im Standard deaktiviert. PRTG und kundenspezifische GitLab-Proxydateien gehoeren nicht zur Collection.

Die Rolle [`kernel_crash_recovery`](roles/kernel_crash_recovery/README.md) konfiguriert die automatische Wiederherstellung nach Kernel-Oops/Panic. Das Collection-Playbook `degitag.infrastructure.kernel_crash_recovery` rollt seriell auf per Inventory-Limit ausgewaehlte Hosts aus.

### Gemeinsame Nutzung

[Gemeinsamer Nutzungsvertrag](docs/collection-usage.md) beschreibt Installation, Rollenaufbau, Abhängigkeiten und den einheitlichen lokalen Prüfaufruf `bash tests/run_checks.sh`.

## Plattformen und Laufzeit

| Zielsystem | Versionen |
| --- | --- |
| Debian | 13 |
| Ubuntu LTS | 22.04, 24.04, 26.04 |
| Rocky Linux / RHEL | 9 |

Ausnahme: `dante` unterstuetzt Ubuntu und EL9; Debian 13 enthaelt kein entsprechendes Paket in Stable.

Die Matrix beschreibt die beabsichtigte Kompatibilitaet. Fachlich distributionsgebundene Rollen sind im jeweiligen README und im Preflight enger begrenzt. Der konkrete Nachweis steht in [Validierung](docs/validation/README.md); ein Metadateneintrag ist kein bestandener Funktionstest. Rocky-Tests ersetzen keinen echten RHEL-Test mit passenden Repositories und Subscription.

Controller: Python 3.12–3.14 und Ansible Core 2.20/2.21. Die verwalteten Hosts verwenden ihre systemeigene Python-Installation. Collection-Abhaengigkeiten und Versionsgrenzen stehen in `galaxy.yml` und `requirements.yml`.

## Installation

```yaml
# requirements.yml
collections:
  - name: git+https://github.com/degitag/ansible-collection-infrastructure.git
    version: v2.1.0
```

Lokal aus dem vorbereiteten Checkout bauen:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-galaxy collection build --force
ansible-galaxy collection install degitag-infrastructure-2.1.0.tar.gz --force
```

Das GitHub-Repository liegt unter `degitag`, die Collection heisst `degitag.infrastructure`. Die Git-Installation setzt einen passenden Release-Tag voraus; der Build aus dem Checkout ist davon unabhaengig.

## Verwendung

```yaml
- name: Configure Linux baseline
  hosts: servers
  become: true
  gather_facts: true
  roles:
    - role: degitag.infrastructure.packages
    - role: degitag.infrastructure.chrony
    - role: degitag.infrastructure.firewall
      vars:
        firewall_rules:
          - port: '22'
            protocol: tcp
            source: 192.0.2.0/24
          - port: '443'
            protocol: tcp
```

Secrets stammen aus Ansible Vault oder dem Betreiber-Secret-Store. Beispiele verwenden ausschliesslich Dokumentationsadressen und keine produktiven Zugangsdaten.

## Rollen

| Rolle | Verantwortung |
| --- | --- |
| [`apache`](roles/apache/README.md) | Install and configure Apache HTTP Server and virtual hosts. |
| [`apt_repos`](roles/apt_repos/README.md) | Manage APT repository definitions and signing keys on Debian-based systems. |
| [`auditd`](roles/auditd/README.md) | Manage additional auditd rules. |
| [`borgbackup`](roles/borgbackup/README.md) | Install and configure BorgBackup jobs and defaults. |
| [`borgmatic`](roles/borgmatic/README.md) | Install and configure borgmatic backup jobs and timers. |
| [`ca`](roles/ca/README.md) | Deploy custom certificate authorities. |
| [`certbot`](roles/certbot/README.md) | Request and renew TLS certificates with Certbot. |
| [`chrony`](roles/chrony/README.md) | Install and configure Chrony for time synchronization. |
| [`dante`](roles/dante/README.md) | Install and configure the Dante SOCKS proxy service. |
| [`dnf_automatic`](roles/dnf_automatic/README.md) | Configure automatic package updates on EL systems. |
| [`dnf_repos`](roles/dnf_repos/README.md) | Manage YUM and DNF repository definitions and signing keys on EL systems. |
| [`docker`](roles/docker/README.md) | Install Docker Engine from the official Docker repositories and manage daemon settings. |
| [`fail2ban`](roles/fail2ban/README.md) | Install and configure Fail2ban jails and actions. |
| [`firewall`](roles/firewall/README.md) | Manage a common firewall rule API through UFW or firewalld. |
| [`gui`](roles/gui/README.md) | Graphical desktop |
| [`haproxy`](roles/haproxy/README.md) | Install and configure HAProxy load balancers and TLS assets. |
| [`hosts`](roles/hosts/README.md) | Manage hostname-related entries in /etc/hosts. |
| [`keepalived`](roles/keepalived/README.md) | Install and configure Keepalived VRRP instances and helper scripts. |
| [`logging`](roles/logging/README.md) | Configure a portable journald and logrotate host logging baseline. |
| [`logrotate`](roles/logrotate/README.md) | Manage custom logrotate rules for application and service logs. |
| [`lvm`](roles/lvm/README.md) | Create and manage LVM volume groups and logical volumes. |
| [`motd`](roles/motd/README.md) | Configure the login message of the day. |
| [`mounts`](roles/mounts/README.md) | Manage filesystem and network mounts. |
| [`msmtp`](roles/msmtp/README.md) | Configure msmtp relay accounts and aliases. |
| [`multipath`](roles/multipath/README.md) | Configure device-mapper multipath on supported Linux systems. |
| [`network`](roles/network/README.md) | Translate a common network API to Netplan, ifupdown or NetworkManager. |
| [`nfs`](roles/nfs/README.md) | Configure NFS server exports and client mounts. |
| [`nginx`](roles/nginx/README.md) | Install and configure NGINX and virtual hosts. |
| [`packages`](roles/packages/README.md) | Install and remove packages from aggregated package lists. |
| [`proxy`](roles/proxy/README.md) | Configure proxy environment files for system tools. |
| [`qemu_guest_agent`](roles/qemu_guest_agent/README.md) | Install the QEMU guest agent package. |
| [`resolvconf`](roles/resolvconf/README.md) | Manage resolver nameservers and search domains. |
| [`sftp_server`](roles/sftp_server/README.md) | Manage chrooted SFTP-only accounts and SSH match configuration. |
| [`squid`](roles/squid/README.md) | Install and configure the Squid HTTP proxy service. |
| [`sshd`](roles/sshd/README.md) | Configure OpenSSH client and server settings. |
| [`sssd`](roles/sssd/README.md) | Configure SSSD for directory integration. |
| [`sysctl`](roles/sysctl/README.md) | Manage sysctl kernel parameters. |
| [`systemd_resolved`](roles/systemd_resolved/README.md) | Configure systemd-resolved DNS settings. |
| [`systemd_timesyncd`](roles/systemd_timesyncd/README.md) | Configure systemd-timesyncd NTP settings. |
| [`telegraf`](roles/telegraf/README.md) | Install and configure Telegraf with plugin definitions. |
| [`traefik`](roles/traefik/README.md) | Install and configure Traefik reverse proxies and routing rules. |
| [`unattended_updates`](roles/unattended_updates/README.md) | Configure unattended upgrades on Debian and Ubuntu. |
| [`unbound`](roles/unbound/README.md) | Install and configure the Unbound DNS resolver and local zones. |
| [`users`](roles/users/README.md) | Manage local users, SSH keys, and shell dotfiles. |
| [`vmware_tools`](roles/vmware_tools/README.md) | Install open-vm-tools on supported Linux systems. |
| [`wpad`](roles/wpad/README.md) | Manage a proxy auto-configuration file and publish it through Apache or NGINX. |
| [`conditional_reboot`](roles/conditional_reboot/README.md) | Explizit aktivierte Wartungsneustarts. |
| [`prompt`](roles/prompt/README.md) | Interaktiver Bash-Prompt und optionale Shell-Helfer. |

## Repository-Struktur

Der [gemeinsame Repository-Standard](docs/repository-standard.md) gilt fuer alle fuenf Collections. `roles/` enthaelt die Rollen, `plugins/` die Filter, `tools/` Generatoren und Vertragspruefer, `tests/unit/` isolierte Pruefungen, `tests/integration/` fachliche Tests und `docs/validation/` datierte Nachweise. Release-Artefakte enthalten keine Controller-Abhaengigkeiten, Tests, Entwicklungsdokumentation oder Caches.

## Entwicklung und Checks

```bash
/opt/homebrew/opt/python@3.12/bin/python3.12 -m venv .venv
make prepare
.venv/bin/python tools/generate_role_scaffold.py
.venv/bin/python tools/format_yaml.py
make ci
```

`make ci` und `make test` rufen `bash tests/run_checks.sh` auf. Dieser Einstieg prueft Repository-Vertrag, Scaffold, Unit-Tests, YAML, Build/Artefakt/Installation, sichere Render-/Preflight-Pruefungen, Syntax, Ansible-Lint und Sanity. `make unit`, `lint`, `syntax`, `build` und `package-smoke` bieten Teilpruefungen. Der lokale Lauf verwendet die installierte Core-Version; CI prueft Core 2.20 und 2.21 unter Python 3.12.

Oeffentliche APIs, Beschreibungen und Beispiele werden in `tools/role_scaffold_config.yml` gepflegt; fachliche Tasks stehen in `tasks/configuration.yml`. Generierte Dateien nicht direkt aendern. Weitere Einrichtung und Review-Regeln stehen in [CONTRIBUTING.md](CONTRIBUTING.md), die aktuelle Testabdeckung in [docs/validation/](docs/validation/README.md).

## CI und Releases

CI ruft denselben vollstaendigen Pruefeinstieg `bash tests/run_checks.sh` mit Core 2.20/2.21 unter Python 3.12 auf. Ansible-Lint verwendet das Profil `production`. Fachliche Smoke-/Functional-Tests bleiben eigene Jobs. Actions sind auf Commit-SHAs festgelegt; Dependabot ueberwacht Updates. Schreibrechte sind auf den separaten Release-Workflow begrenzt.

Netzwerk, Firewall, SELinux, Dienststarts und Neustarts werden auf dedizierten Test-VMs geprueft. Ein produktiver Rollout ist kein Bestandteil der Tests. Releases entstehen erst bei einem explizit gepushten Versions-Tag; keine automatische Galaxy-Veroeffentlichung.

## Migration

Siehe [Migration](docs/migration.md) fuer den neuen Namespace, die entfernten Backend-Rollen und die Aufteilung der Collections.

## Sicherheit

Secrets und vertrauliche Meldungen sind in [SECURITY.md](SECURITY.md) beschrieben. Beispiele verwenden Vault-Referenzen und Dokumentationsadressen. Betriebseingriffe und Releases werden separat freigegeben.

## Lizenz

[MIT](LICENSE).
