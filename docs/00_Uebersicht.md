# Übersicht

> Ein Lern-Lab für eine fiktive Firma: sieben virtuelle Systeme mit DNS, Webportal, Mail, Backup und Überwachung. Am Ende wurde geprüft, ob alles belegbar funktioniert.

## Ziel

Diese Dokumentation beschreibt den Aufbau einer virtuellen Unternehmensumgebung für ein fiktives Unternehmen („Fraîche Flow“, Bewässerung und Sprühnebel für eine Stadt): Netz, Namensdienste, Speicher, Webplattform, Container, Mailserver, Backup, Überwachung und eine technische Abnahme. Es ist ein Lernprojekt im Rahmen der Umschulung zum Fachinformatiker für Systemintegration und ein Testlabor, kein Produktivbetrieb.

Jede Seite folgt demselben Aufbau, damit sich die Schritte nachvollziehen und wiederholen lassen:

| Abschnitt | Inhalt |
|---|---|
| Ziel | Was erreicht werden soll und warum |
| Grundlagen | Die nötige Theorie in kurzer Form, mit erklärten Fachbegriffen |
| Meine Umsetzung | Die Schritte mit Befehlen und Konfiguration, dazu Ausschnitte als Nachweis |
| Ergebnis | Was funktioniert und was nicht getestet wurde |
| Worauf zu achten ist | Fallstricke. Gekennzeichnet ist, was im Lab wirklich aufgetreten ist |
| Dateien | Verweise auf Konfigurationen und Skripte |

## Grundlagen

Das Lab läuft auf einem Windows-Host mit Hyper-V. Darin liegen vier virtuelle Maschinen: Router, Management-Client, Proxmox-Server und TrueNAS. Proxmox betreibt wiederum zwei VMs und einen LXC-Container. Alle Systeme liegen im Netz `192.168.10.0/24` mit der Domain `fraicheflow.test`. Der Übergang nach außen läuft nur über ff-edge-01.

| Host | IP | Aufgabe |
|---|---|---|
| ff-edge-01 | 192.168.10.1 | Router, NAT, DNAT |
| ff-pve-01 | 192.168.10.10 | Proxmox VE (VMs core und app, LXC mail) |
| ff-core-01 | 192.168.10.11 | DNS (Bind9), DHCP (Kea) |
| ff-store-01 | 192.168.10.12 | TrueNAS, iSCSI |
| ff-app-01 | 192.168.10.13 | Nextcloud, Docker, Backup |
| ff-mail-01 | 192.168.10.14 | Mailserver |
| ff-mgmt-01 | 192.168.10.50 | Administration, Git, Uptime Kuma |

Technischer Schwerpunkt:

- **Virtualisierung:** Hyper-V als Unterbau, darin Proxmox VE mit zwei VMs und einem LXC-Container.
- **Basisdienste:** DNS (Bind9), DHCP (Kea), Router mit NAT und DNAT.
- **Speicher:** TrueNAS mit iSCSI, getrennte Targets für Daten und Backup.
- **Anwendungen:** Apache, Nextcloud, MariaDB, Portainer, chrony, Mailserver.
- **Betrieb:** Backup mit Prüfung und Alarm, Restore-Test, Monitoring, SSH-Härtung, Git.

## Meine Umsetzung

Meine Rolle war Einrichtung, Konfiguration, Tests, Fehlersuche und Dokumentation. Die Seiten sind nach dem Aufbau geordnet:

| Nr. | Thema | Inhalt |
|---|---|---|
| 01 | [Architektur](01_Architektur.md) | Netz, Systeme, Rollen, Hyper-V und Proxmox |
| 02 | [Management und Git](02_Management_Git.md) | Administrationsclient, Versionierung |
| 03 | [Router, NAT und DNAT](03_Edge_NAT_DNAT.md) | ff-edge-01, Portweiterleitung |
| 04 | [DNS mit Bind9](04_DNS_Bind9.md) | Zonen, Prüfung mit `dig` |
| 05 | [DHCP mit Kea](05_DHCP_Kea.md) | Adressvergabe |
| 06 | [SSH-Schlüssel und Alias](06_SSH_Schluessel.md) | Schlüsselpaar, Passphrase, Alias |
| 07 | [Passwort-Anmeldung abschalten](07_SSH_Passwort_abschalten.md) | Drop-in, Gegenprobe |
| 08 | [SSH auf allen Servern](08_SSH_Alle_Server.md) | Alle fünf Server, Abschlussprüfung |
| 09 | [TrueNAS und iSCSI](09_TrueNAS_iSCSI.md) | Speicher für Proxmox |
| 10 | [Backup-Ziel per iSCSI](10_Backup_Ziel.md) | `target02`, Mount von Hand |
| 11 | [Backup-Skript](11_Backup_Skript.md) | Nächtliche Sicherung mit Prüfung |
| 12 | [Backup-Alarm](12_Backup_Alarm.md) | Mail bei Fehler, Push bei Erfolg |
| 13 | [Container-Backup und Restore-Test](13_Container_Backup_Restore.md) | vzdump, Wiederherstellung |
| 14 | [Nextcloud mit Apache](14_Nextcloud_Apache.md) | VHosts, MariaDB, PHP, TLS |
| 15 | [Docker und Compose](15_Docker_Compose.md) | Engine, Compose, feste Image-Stände |
| 16 | [Portainer mit HTTPS-Proxy](16_Portainer_Proxy.md) | Verwaltungsoberfläche, Reverse Proxy |
| 17 | [Zeitserver chrony](17_chrony.md) | NTP im Container |
| 18 | [Mailserver](18_Mailserver.md) | LXC, docker-mailserver, Thunderbird |
| 19 | [Monitoring mit Uptime Kuma](19_Monitoring_Kuma.md) | Prüfungen, Alarm, Push |
| 20 | [Technische Abnahme](20_Abnahme.md) | Nachweismatrix, Auflagen |
| 21 | [Troubleshooting](21_Troubleshooting.md) | Fehlerbild, Ursache, Lösung |

## Ergebnis

Die technische Abnahme endete mit Auflagen: 14 von 15 Anforderungen sind erfüllt, eine ist teilweise erfüllt.

| Bereich | Stand |
|---|---|
| Dienste (DNS, Web, Container, Mail) | Ende zu Ende geprüft |
| Backup | Nächtlich, mit Prüfung und Alarm |
| Wiederherstellung | Container getestet, VM-Restore offen |
| Sicherheit | SSH nur mit Schlüssel, gemeinsame Administratorkonten offen |
| Überwachung | Zwölf Prüfungen mit Mailalarm |

Offen sind ein getesteter VM-Restore, persönliche Administratorkonten, eine eigene Zertifizierungsstelle und ein Git-Remote für die Konfigurationen. Als Restrisiko bleibt der Speicher als Einzelpunkt. Die Details stehen in der [Abnahme](20_Abnahme.md).

## Worauf zu achten ist

| Punkt | Hintergrund |
|---|---|
| Kein Produktivbetrieb | Das Lab nutzt selbstsignierte Zertifikate, gemeinsame Konten und einen einzelnen Speicher |
| Keine Zugangsdaten | Passwörter, private Schlüssel und Tokens stehen nicht in den Seiten und nicht im Repository |
| Nachweise | Terminalausgaben und Konfigurationen. Was nicht getestet wurde, steht auf den Seiten unter „Ergebnis“ |

## Dateien

Die Konfigurationsdateien und Skripte liegen im Lab und sind nicht als Dateien abgelegt. Ihre wichtigen Teile stehen auf den Themenseiten. Geheimnisse (Passwörter, Schlüssel, Tokens) sind bewusst nicht enthalten.
