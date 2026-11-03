# AGENTS.md

Dieses Repository pflegt die Ansible Collection `degitag.infrastructure`. Dokumentation und Architekturentscheidungen werden aus Sicht eines DevOps- und Systemarchitektur-Teams geschrieben. In Anleitungen `vim` verwenden.

## Verantwortungsgrenze

Die Collection verwaltet Linux-Betriebssysteme, Baseline, allgemeine Dienste und Host-Integrationen. Die sieben `k3s_*`-Rollen liegen im separaten Repository `ansible-collection-k3s` unter `degitag.k3s`. Kubernetes-Anwendungen, Datenmigrationen und Proxmox-Cluster-Backups gehoeren nicht zur Host-Baseline.

Bestehende uncommittete Aenderungen erhalten. Keine produktiven Deployments, Datenmigrationen oder Repository-Transfers aus einer Refactoring- oder Validierungsaufgabe ableiten. Dedizierte Testgaeste duerfen im ausdruecklich freigegebenen Testumfang verwendet werden.

## Support-Matrix

- Debian 13
- Ubuntu 22.04, 24.04 und 26.04
- Rocky Linux 9 und RHEL 9

Metadaten, README, Variablen und Preflight muessen dieselbe Matrix beschreiben. RHEL setzt zugaengliche, abonnierte Basis-Repositories voraus. Ein Rocky-Test ist kein echter RHEL-Test.

Familiengebundene Rollen bleiben bewusst eingegrenzt:

- Debian/Ubuntu: `apt_repos`, `systemd_resolved`, `systemd_timesyncd`, `unattended_updates`.
- Rocky/RHEL: `dnf_repos`, `dnf_automatic`.

`dante` ist distributionsgebunden auf Ubuntu und EL9: Debian 13 bietet kein Dante-Paket in Stable. Alle anderen Rollen haben ein portables Ziel. Neue Distributionen oder Releases werden erst nach Anpassung von Matrix, Assertions, Plattformdateien und Tests freigegeben.

## Verbindliches Rollen-Schema

Jede Rolle enthaelt:

- `defaults/main.yml`: oeffentliche Eingaben ohne Secrets.
- `tasks/main.yml`: generierter Einstieg mit Preflight und Import von `configuration.yml`.
- `tasks/preflight.yml`: generierte Fact- und Plattformpruefung.
- `tasks/configuration.yml`: fachliche Implementierung; weitere Taskdateien nach Verantwortungsbereich.
- `vars/platform/<Familie>.yml`, `<Distribution>.yml`, `<Distribution>-<Version>.yml`.
- `vars/main.yml` fuer gemeinsame interne Konstanten, falls erforderlich.
- `handlers/main.yml`, wenn Aenderungen einen Handler benoetigen.
- `meta/main.yml`, `meta/argument_specs.yml`, `README.md`, `tests/test.yml`.

Der Einstieg sammelt fehlende benoetigte Facts und verwirft nicht unterstuetzte Releases vor Aenderungen. Vorhandene Facts werden wiederverwendet. Optionale IPv4-/IPv6-Routen duerfen auf einem Host fehlen. Beispiele und Tests sammeln Facts auf Play-Ebene. Neue Implementierungen verwenden `ansible_facts` statt injizierter globaler Fact-Aliase.

Die Plattformschicht laedt Familie, Distribution und Release in dieser Reihenfolge. Leere Dateien bedeuten geerbte Werte. OS-abhaengige Paketnamen, Dienstnamen, Pfade, Validatoren und SELinux-Werte gehoeren in diese Schicht; keine Distributionsannahmen in Templates verstecken.

Oeffentliche Defaults muessen auch bei der automatischen Argumentvalidierung vor dem Preflight auswertbar sein. OS-abgeleitete optionale Defaults brauchen deshalb einen typgerechten Fallback. Der Fallback darf die spaetere effektive Plattformauswahl nicht festschreiben.

## Rollen-API und Defaults

- `meta/argument_specs.yml` ist der verbindliche, typisierte Variablenvertrag.
- Jede oeffentliche Defaultvariable wird dokumentiert und typisiert. Strukturelle Listen brauchen bei neuen APIs Unterfelder, Typen und Choices.
- Defaults erlauben einen sicheren Rollout oder erzeugen einen klaren Fehler fuer fachlich notwendige Pflichtdaten. Keine Platzhalter-Passwoerter, produktiven Zieladressen oder nicht dokumentierten Fremdvariablen.
- Pflichtdaten wie SMTP-Relay, Verzeichnis-Domain, Cluster-Token oder Migrationsquelle werden vor dem Eingriff validiert.
- Repository-Initialisierung, Backup-Zeitplaene, statische Resolver-Uebernahme und optionale Fremdbinaries brauchen ausdrueckliche Schalter.
- Paket- und Konfigurationsdefaults muessen durch Inventory-Eingaben sinnvoll uebersteuerbar bleiben.

## Netzwerk und Firewall

`network` ist der einzige oeffentliche Einstieg fuer Host-Netzwerk:

- Ubuntu: Netplan mit networkd.
- Debian: ifupdown; deklarierte DNS-Daten ueber den nativen Resolver.
- Rocky/RHEL: NetworkManager mit nmcli.

`firewall` ist der einzige oeffentliche Einstieg fuer Portregeln:

- Debian/Ubuntu: UFW.
- Rocky/RHEL: firewalld.

Die frueheren oeffentlichen Rollen `ufw`, `firewalld`, `netplan`, `interfaces`, `ifcfg` und `nmcli` entfallen in Version 2. Dienstrollen delegieren Portregeln ueber `include_role` an `degitag.infrastructure.firewall`; sie rufen die Backendmodule nicht selbst auf.

Fuer jede deklarierte Schnittstelle gilt genau eine Verantwortlichkeit. Bestehende Manager oder fremde Profile nicht stillschweigend loeschen. NetworkManager verwendet vorhandene aktive Profile, statt konkurrierende Definitionen anzulegen. Backend-Migrationen werden ausdruecklich freigegeben. Aktivierung wird getrennt vom Rendern bzw. nativen Check-Modus gesteuert. Ein unterbrochener Lauf muss auch bei bereits geschriebener Konfiguration wieder konvergieren.

Firewall-Regeln sind additiv. Eine Dienstrolle setzt keine globale Policy zurueck. Vollstaendig vertraute Netze und Forwarding sind eine eigene ausdrueckliche Entscheidung. Entfernen von Regeln muss nachvollziehbar und auf eigene Regeln begrenzt bleiben.

## Dienststeuerung, Security und SELinux

- Dienstrollen bieten soweit sinnvoll `*_manage_service`, `*_service_enabled`, `*_service_state`.
- Diese Schalter gelten auch fuer Restart-/Reload-Handler. Ein Konfigurationswechsel darf einen bewusst gestoppten Dienst nicht starten.
- Native Konfiguration vor Aktivierung validieren, wenn ein geeigneter Validator vorhanden ist.
- SSH, SFTP und SSSD besonders sorgfaeltig pruefen; erfolgreiche Syntax ist noch kein erfolgreicher Zugriff.
- SFTP-Schluessel pro Benutzer in getrennten, rootverwalteten Dateien halten. Ein Homepfad innerhalb eines Chroots ist kein gemeinsamer Hostpfad fuer `authorized_keys`.
- Unterstuetzte EL9-Dienste verwalten notwendige SELinux-Pakete, Ports, Kontexte und Booleans selbst. Enforcement nicht abschalten.
- Den von der Distribution unterstuetzten Dienststeuerungsweg verwenden, insbesondere fuer auditd unter EL9.
- Sensible Konfiguration mit restriktiven Dateirechten und `no_log` schreiben.
- Externe Repositories und Downloads dokumentieren. Signaturen/Fingerprints pruefen, Binaries und Bootstrap-Installer moeglichst nach Version und Checksumme festlegen.

## Idempotenz und Bereinigung

Generierte Dateien sind deterministisch. Laufzeit-Timestamps gehoeren nicht in Templates, wenn sie jeden Lauf aendern. Abgeleitete Listen vor erneutem Aufbau initialisieren.

Veraltete eigene Artefakte dort entfernen, wo die Rolle die Quelle der Wahrheit ist. Fremde Dateien, Repositories, Netzwerkprofile und andere Betreiberkonfiguration nicht ungefragt entfernen. Storage-Rollen duerfen keine destruktive Rekonfiguration aus leeren Defaults ableiten.

## Generator und lokale Toolchain

Geruest und generierte Inhalte werden ueber `tools/generate_role_scaffold.py` und `tools/role_scaffold_config.yml` gepflegt. Beschreibungen, Betreiberhinweise, Beispiele und explizite Argumentvertraege gehoeren in die Konfiguration. Fachliche Tasks stehen in `tasks/configuration.yml`, nicht im generierten Einstieg.

Auf macOS eine eigene `.venv` mit Homebrew Python 3.12 verwenden. Homebrew ohne sudo betreiben:

```bash
/opt/homebrew/opt/python@3.12/bin/python3.12 -m venv .venv
.venv/bin/python -m pip install -r requirements-test.txt
.venv/bin/ansible-galaxy collection install -r requirements.yml
```

Nach Struktur-Aenderungen:

```bash
.venv/bin/python tools/generate_role_scaffold.py
.venv/bin/python tools/format_yaml.py
```

## Verpflichtende Pruefungen

- Generator mit `--check`.
- `pytest tests/unit`.
- YAML-Lint und Ansible-Lint mit Profil `production`.
- Syntaxpruefung jeder Rolle mit `gather_facts: true`.
- Collection bauen, installieren und Artefakt auf Identitaet sowie ausgeschlossene Caches/Secrets pruefen.
- `ansible-test sanity` fuer Compile und Import ausserhalb eines von Git ignorierten Installationsverzeichnisses ausfuehren.
- Container-Smoke-Tests auf allen fuenf frei verfuegbaren Zielimages, einschliesslich zweitem Lauf mit `changed=0`.
- Vorhandene Functional-Tests bei betroffenen Rollen oder gemeinsamen Aenderungen ausfuehren.

Netzwerk, Firewall, Dienststarts, SELinux und Neustarts brauchen zusaetzlich echte dedizierte VMs. Container ersetzen diese Pruefungen nicht. Testumfang und verbleibende Luecken in `docs/validation/` festhalten; keine pauschale Abnahme aller Features aus einem gruenen Syntaxlauf ableiten.

CI-Actions nach Commit-SHA referenzieren, minimale Berechtigungen setzen und Abhaengigkeiten mit Dependabot pflegen. Releases erst nach passenden Versions-Tags und bestandenen Checks; keine implizite Galaxy-Veroeffentlichung.

## Dokumentation und Zugangsdaten

Collection-Referenzen immer als `degitag.infrastructure.*` schreiben. K3s-Referenzen verwenden `degitag.k3s.*`. Beispiele enthalten Dokumentationsadressen und Secret-Store-Referenzen.

README, Migrationsvertrag und technische Validierungsnachweise werden im Repository versioniert. Betriebsdokumentationen werden in Outline gepflegt, wenn dies Teil des Auftrags ist. API-Zugangsdaten liegen gegebenenfalls in `~/.zshenv`; nur die benoetigten Variablen verwenden und niemals ihren Inhalt ausgeben oder ins Repository kopieren.

## Änderungsvertrag und Abnahme

- Wiederverwendbare Rollen und Module pflegen; kundenspezifische Hostwerte, Ziele und Secrets gehören in nutzende Inventories beziehungsweise deren Secret-Store.
- Öffentliche Variablen dokumentieren, sinnvoll und sicher vorbelegen sowie typisiert validieren. Fachlich erforderliche Eingaben vor Eingriffen prüfen. Bei kompatiblen Änderungen bestehende Nutzer und Variablenverträge erhalten; notwendige Inkompatibilitäten ausdrücklich mit Migrationsweg dokumentieren.
- Änderungen gegen die deklarierte Distributions- und Ansible-Supportmatrix aus README, Metadaten und Preflight prüfen; keine Unterstützung allein aus einem Syntaxlauf ableiten. Rollenbezogene Einschränkungen bleiben bestehen.
- Idempotenz auf geeigneten Testsystemen durch einen unveränderten zweiten Lauf mit `changed=0` nachweisen. Check-Modus soweit unterstützt berücksichtigen und seine Grenzen benennen. Echtes Dienstverhalten einschließlich systemd/journald auf geeigneten Systemen prüfen; Container allein belegen diese Funktion nicht.
- Den gemeinsamen Prüfeinstieg `bash tests/run_checks.sh` für Implementierungsänderungen und die zur Änderung passenden zusätzlichen Tests verwenden. Bei reinen Arbeitsanweisungsänderungen Markdown, Diff und Konsistenz prüfen und die ausgelöste erforderliche CI vollständig abwarten; keine künstlichen Implementierungs- oder Hosttests dafür anlegen.
- Changelog und Version passend zu funktionaler Änderung und Kompatibilität pflegen. Collection bauen und Artefakt prüfen; bei tatsächlichen Releases die Release-Prüfungen bis zum erfolgreichen Endzustand verfolgen. Reine Anweisungsänderungen begründen keinen Release-Tag. Eine Collection-Veröffentlichung autorisiert keinen Rollout auf nutzende Systeme.

## Auftragsabschluss und Dokumentation

Der globale Abschlussstandard gilt auch hier: Eigene Änderungen committen, pushen und per Pull Request (bei GitLab Merge Request) nach erfolgreichen erforderlichen Prüfungen integrieren; auftragsrelevante Nach-Merge-Prüfungen abwarten und erst danach eigene integrierte Arbeitsbranches bereinigen. Bestehende unabhängige Änderungen, Branches und Worktrees erhalten. Schutzregeln und erforderliche Reviews nicht umgehen; echte Blocker mit offenem Schritt benennen.

Repository-Integration, Veröffentlichung und produktiver Betrieb sind getrennte Ergebnisse. Eine erfolgreiche Prüfung oder ein Merge erteilt keine zusätzliche Produktionsfreigabe. Bestehende konkrete Freigaberegeln bleiben verbindlich; eine pauschale automatische Produktionsfreigabe wird hier nicht eingeführt.

In Anleitungen `vim` verwenden. Dokumentation aus Sicht von DevOps und Systemarchitektur schreiben. Technische Nachweise im Repository und bestehende Betriebsdokumentation in Outline erhalten; Outline nur im beauftragten Umfang bearbeiten. Secrets aus geschützten Variablenquellen beziehen, niemals ausgeben oder ins Repository übernehmen.
