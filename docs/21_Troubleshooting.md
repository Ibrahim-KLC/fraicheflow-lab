# Troubleshooting

> Fehler, die im Lab wirklich vorkamen, mit Ursache, Lösung und einer Prüfung, an der man erkennt, dass der Fehler weg ist.

## Ziel

Die Seite sammelt die Fehler, die im Lab tatsächlich aufgetreten sind, jeweils mit Fehlerbild, Ursache, Lösung und einer Prüfung, an der man erkennt, dass der Fehler behoben ist. Sie zeigt außerdem eine feste Reihenfolge für die Fehlersuche. Ursachen sind nur dort genannt, wo sie belegt sind. Wo die Ursache nicht geklärt wurde, steht das ausdrücklich da.

## Grundlagen

Ein Dienst erreicht den Benutzer über mehrere Schichten: Name, Netz, Port, Dienst, Anwendung. Ein Fehler steckt meist in genau einer davon. Deshalb wird von unten nach oben geprüft, und zuerst wird der tatsächliche Zustand festgestellt, dann erst die Ursache vermutet.

| Schicht | Typischer Befehl | Zeigt |
|---|---|---|
| Namensauflösung | `dig @192.168.10.11 name.fraicheflow.test +short` | Ob der Name auf die richtige IP zeigt |
| Erreichbarkeit | `ping -c 2 192.168.10.13` | Ob der Host antwortet |
| Port | `nc -zv 192.168.10.14 25` | Ob der Dienst Verbindungen annimmt |
| Anwendung | `curl -skI https://nextcloud.fraicheflow.test/` | Ob die Anwendung richtig antwortet |
| Dienststatus | `systemctl status <dienst>` oder `docker ps` | Ob der Prozess läuft |

Ein Beispiel: Ein Webdienst ist nicht erreichbar. `dig` zeigt, ob der Name stimmt. Stimmt er, prüft `ping` die Adresse, dann `nc` den Port und zuletzt `curl` die Anwendung. Der erste Schritt, der scheitert, grenzt den Fehler ein.

## Meine Umsetzung

Der Fehlerkatalog ist nach Themen sortiert. Die Spalte „Seite“ verweist auf die ausführliche Beschreibung.

### Namensauflösung und DHCP

| Fehlerbild | Ursache | Lösung | Prüfung | Seite |
|---|---|---|---|---|
| `named-checkzone` meldet `CNAME and other data` | Ein Name hatte zugleich einen `A`- und einen `CNAME`-Eintrag | Pro Name nur einen Eintrag. `portal` und `nextcloud` als CNAME, Serial erhöht | `dig` liefert 192.168.10.13 | [04](04_DNS_Bind9.md) |
| `named-checkzone` meldet `unknown RR type` | In einer Zeile fehlte der Record-Typ `A` | Typ ergänzt | `named-checkzone` ohne Fehler | [04](04_DNS_Bind9.md) |
| `rndc reload` meldet Erfolg, die Zone ist trotzdem nicht geladen | `rndc reload` meldet auch dann Erfolg, wenn die Zone wegen eines Fehlers nicht geladen wurde | Vor dem Neuladen `named-checkzone` ausführen | Abfrage mit `dig` zeigt den neuen Eintrag | [04](04_DNS_Bind9.md) |
| Name löst auf ff-core-01 auf, nicht auf ff-mgmt-01 (`Could not resolve host`) | Der Resolver auf ff-mgmt-01 hatte die negative Antwort gespeichert | `resolvectl flush-caches` | `getent hosts nextcloud.fraicheflow.test` liefert 192.168.10.13 | [04](04_DNS_Bind9.md), [14](14_Nextcloud_Apache.md) |
| Interne Namen werden auf ff-mgmt-01 nicht aufgelöst | Der Client fragte von Hand eingetragen `1.1.1.1` | DNS-Server im NetworkManager-Profil auf 192.168.10.11 gesetzt | `resolvectl status` zeigt 192.168.10.11 | [04](04_DNS_Bind9.md) |
| Kea startet nach einer Änderung nicht (`failed`) | In der JSON-Datei fehlte ein Komma zwischen zwei Einträgen | Einträge getrennt geschrieben, Komma ergänzt, Dienst neu gestartet | `systemctl status kea-dhcp4-server` zeigt `active (running)` | [17](17_chrony.md), [05](05_DHCP_Kea.md) |
| Kea lauscht auf keiner Karte | Der Kartenname in der Konfiguration enthielt ein Leerzeichen und stimmte nicht mit `ens18` überein | Namen exakt aus `ip -br a` übernommen | Im Journal erscheint `DHCP4_STARTED` | [05](05_DHCP_Kea.md) |
| `kea-dhcp4 -t` meldet `Unable to open file` | Nicht geklärt, die Datei war lesbar | Als Nachweis gilt der laufende Dienst | `journalctl -u kea-dhcp4-server` mit `DHCP4_STARTED` | [05](05_DHCP_Kea.md) |

### SSH

| Fehlerbild | Ursache | Lösung | Prüfung | Seite |
|---|---|---|---|---|
| Schlüssel wird abgelehnt, obwohl er kopiert wurde | In `authorized_keys` fehlte am Zeilenanfang der Schlüsseltyp `ssh-ed25519` | Zeile vollständig mit Typ, Schlüssel und Kommentar anlegen | Anmeldung mit Schlüssel gelingt | [08](08_SSH_Alle_Server.md) |
| `ssh-copy-id` auf ff-pve-01 nimmt das Root-Passwort nicht an | Nicht untersucht | Schlüssel über die Web-Shell von Proxmox eingetragen | `ssh root@192.168.10.10 whoami` liefert `root` | [08](08_SSH_Alle_Server.md) |
| Passwort-Anmeldung auf mehreren Servern offen | Standardeinstellung von `sshd` | Drop-in-Datei mit `PasswordAuthentication no` | Gegenprobe liefert `Permission denied (publickey)` | [07](07_SSH_Passwort_abschalten.md) |
| Zweite Datei `00-hardening.con` im Verzeichnis | Tippfehler im Dateinamen | Datei gelöscht | `ls /etc/ssh/sshd_config.d/` | [08](08_SSH_Alle_Server.md) |
| `sshd -T` auf ff-mail-01: `Missing privilege separation directory: /run/sshd` | Nicht untersucht | `sshd -t` ohne Ausgabe und Gegenprobe von ff-mgmt-01 als Nachweis | Gegenprobe liefert `Permission denied (publickey)` | [08](08_SSH_Alle_Server.md) |

### Speicher und Backup

| Fehlerbild | Ursache | Lösung | Prüfung | Seite |
|---|---|---|---|---|
| Zwei Nächte kein Backup, kein Hinweis | Die TrueNAS-VM lief nicht. Der Hyper-V-Host hatte zu wenig freien Arbeitsspeicher (`0x8007000E`) | Arbeitsspeicher von ff-pve-01 von 6144 auf 5198 MB verringert, Alarm und Überwachung eingeführt | Backup läuft, Kuma zeigt `Up` | [09](09_TrueNAS_iSCSI.md), [12](12_Backup_Alarm.md) |
| Log mit `connection timed out` und `mount … failed` | Das Ziel war nicht erreichbar | `mountpoint -q` vor dem Kopieren | Nur bei erfolgreichem Mount läuft `rsync` | [11](11_Backup_Skript.md) |
| Kernel meldet `device offline` und `Buffer I/O error` für `sdb1` | Das iSCSI-Gerät verschwand im laufenden Betrieb | Erst aushängen, nur bei Erfolg abmelden | Lauf endet mit `ENDE (getrennt)` | [11](11_Backup_Skript.md) |
| Vergleich meldet 740 Abweichungen | Der Vergleich nutzte andere Optionen (`-rcn`) als die Kopie | Auf `-aicn` angepasst | Neuer Lauf: `Abweichungen: 0` | [11](11_Backup_Skript.md) |
| Skript läuft in der Shell, aber nicht in cron | cron hat nur `PATH=/usr/bin:/bin` | Test mit `env -i PATH=/usr/bin:/bin` vor dem Einrichten | `exit=0` mit reduzierter Umgebung | [11](11_Backup_Skript.md) |
| `sudo skript >> /var/log/backup.log` scheitert mit `Permission denied` | Die Umleitung `>>` führt die Shell des Benutzers aus, nicht `sudo` | In der Crontab von `root` einrichten | Der Eintrag schreibt ins Log | [11](11_Backup_Skript.md) |
| Alarmmail wird abgelehnt: `Helo command rejected: need fully-qualified hostname`, `curl` zeigt nur Exit 55 | Der Mailserver verlangt einen vollständigen Rechnernamen beim HELO | Vollständigen Namen an die SMTP-URL angehängt | Mail im Postfach `alerts@` | [12](12_Backup_Alarm.md) |
| Kein Alarm, wenn das Backup gar nicht läuft | Der Wrapper kann nur Fehler melden, kein Ausbleiben | Push-Monitor mit 25-Stunden-Fenster | Test mit 60 Sekunden: `DOWN` und Mail | [12](12_Backup_Alarm.md), [19](19_Monitoring_Kuma.md) |
| Proxmox warnt beim Container-Backup zur Thin-Pool-Größe | Zugesagte Größe der virtuellen Platten übersteigt den physischen Platz | Dokumentiert, Belegung wird beobachtet | `lvs` zeigt die Auslastung | [13](13_Container_Backup_Restore.md) |

### Web, Container und Edge

| Fehlerbild | Ursache | Lösung | Prüfung | Seite |
|---|---|---|---|---|
| Falsche Seite erscheint | Die Anfrage trifft den Standard-VHost | Name im DNS eintragen, mit `apache2ctl -S` prüfen | `curl` liefert die erwartete Antwort | [14](14_Nextcloud_Apache.md) |
| `curl` meldet `self-signed certificate` | Selbstsigniertes Zertifikat | `curl -k` im Test, `Ignore TLS/SSL errors` im Monitor | Aufruf mit `-k` liefert `302` | [14](14_Nextcloud_Apache.md) |
| Portainer-Einrichtung gesperrt, Feld „Setup token“ | Portainer sperrt eine neue Installation nach 5 Minuten | `docker restart portainer`, neues Token aus dem Log holen | Anmeldung und Container-Liste | [16](16_Portainer_Proxy.md) |
| Portainer nur über HTTP erreichbar | Es gab nur einen VHost ohne TLS | VHost mit TLS und `ProxyPass` ergänzt | HTTPS-Abruf liefert `200` | [16](16_Portainer_Proxy.md) |
| Portainer-Zugang unbekannt | Erstes Benutzerkonto war nicht mehr verfügbar | Zugang mit `portainer/helper-reset-password` zurückgesetzt | Anmeldung gelingt | [16](16_Portainer_Proxy.md) |
| VM ff-pve-01 startet nicht (`0x8007000E`) | Der Hyper-V-Host hatte zu wenig freien Arbeitsspeicher | Nicht benötigte VM heruntergefahren | ff-pve-01 läuft mit 6144 MB | [15](15_Docker_Compose.md) |
| `:latest` ändert sich unbemerkt | Kein fester Stand des Images | Digest in der Compose-Datei | `docker image inspect` zeigt den Digest | [15](15_Docker_Compose.md) |
| Benutzer `student` konnte über die Gruppe `docker` Root-Rechte erlangen | Mitglieder der Gruppe steuern den Docker-Daemon | Benutzer aus der Gruppe entfernt, `sudo docker` | `getent group docker` ohne Mitglieder | [15](15_Docker_Compose.md) |
| DNAT-Regeln nach Neustart weg | Regeln nur im laufenden Speicher | `netfilter-persistent save` | Regeln in `rules.v4` | [03](03_Edge_NAT_DNAT.md) |
| `iptables -t nat -L -n -V` zeigt keine Regeln | `-V` zeigt die Version, `-v` die ausführliche Ausgabe | Kleinbuchstaben verwenden | Ausgabe mit Zählern | [03](03_Edge_NAT_DNAT.md) |

### Mail und Monitoring

| Fehlerbild | Ursache | Lösung | Prüfung | Seite |
|---|---|---|---|---|
| Port 25 im LXC-Container belegt | Der Postfix von Debian lauscht lokal | `systemctl stop postfix`, `disable postfix` | `ss -tulpen` zeigt nur SSH | [18](18_Mailserver.md) |
| Mailserver beendet sich nach 120 Sekunden | Es existierte noch kein Konto | Konten direkt nach dem Start angelegt | `setup email list` zeigt drei Postfächer | [18](18_Mailserver.md) |
| Thunderbird meldet beim Senden ein selbstsigniertes Zertifikat | Das Zertifikat ist selbstsigniert | Sicherheitsausnahme für `mail.fraicheflow.test:587`. Der Import als CA scheiterte an `basic constraints` | Versand und Antwort funktionieren | [18](18_Mailserver.md) |
| Alarmmail kommt nicht an | Das Postfach `alerts@` fehlte | Postfach angelegt | `doveadm mailbox status` zeigt Mails | [18](18_Mailserver.md) |
| Nextcloud-Monitor `DOWN`: `self-signed certificate` | Selbstsigniertes Zertifikat | `Ignore TLS/SSL errors` im Monitor | Monitor `Up` | [19](19_Monitoring_Kuma.md) |
| Push-Adresse enthält `localhost` | Kuma zeigt seine eigene Adresse | Durch `192.168.10.50` ersetzt | Testlauf meldet `Up` | [19](19_Monitoring_Kuma.md) |

### Git und Grundeinrichtung

| Fehlerbild | Ursache | Lösung | Prüfung | Seite |
|---|---|---|---|---|
| Die E-Mail-Adresse des Autors fehlt, obwohl sie gesetzt wurde | `user.mail` statt `user.email` geschrieben. Git prüft Schlüsselnamen nicht | Mit `--unset` entfernt und richtig gesetzt | `git config --global --list` | [02](02_Management_Git.md) |
| `git log --oneline -all` liefert nicht den erwarteten Graph | Lange Optionen brauchen zwei Bindestriche, sonst liest Git die Buchstaben einzeln | `--all` | Graph erscheint | [02](02_Management_Git.md) |
| Hostname `ff-mgmt-01.ff-mgmt-01.fraicheflow.test` | `sed` ohne das Flag `g` ersetzt nur das erste Vorkommen | `s/debian/ff-mgmt-01/g` | `hostname -f` | [02](02_Management_Git.md) |

## Ergebnis

Der Katalog enthält die Fehler des Labs mit der jeweiligen Lösung und einer Prüfung. Einzelne Ursachen sind offen und als „nicht geklärt“ gekennzeichnet: `kea-dhcp4 -t`, die Meldung von `sshd -T` auf ff-mail-01 und die Ablehnung des Root-Passworts durch `ssh-copy-id` auf ff-pve-01.

## Worauf zu achten ist

| Punkt | Hintergrund |
|---|---|
| Erst Zustand, dann Ursache | Ein Fehler wird zuerst beobachtet und belegt. Erst danach wird eine Ursache angenommen |
| Zeitstempel in Mails | Sie stehen in UTC und liegen im Winter eine Stunde hinter der Berliner Zeit |
| Betreffzeilen in Rohdaten | Sie erscheinen kodiert (`=?UTF-8?Q?...?=`). Das ist das normale MIME-Format, Mailprogramme zeigen es dekodiert |
| Zugangsdaten | Sie gehören nie in Skripte oder Repositorys. Das Backup-Skript liest die Push-Adresse aus `/etc/backup-alarm.conf` (Rechte `600`) |
| Fehlermeldungen vollständig lesen | Die Meldung des Mailservers stand im Log, `curl` zeigte nur den Exit-Code 55 |
| Eine zweite Sitzung offen halten | Bei Änderungen an SSH oder Netzwerk bleibt so ein Rückweg erhalten |

## Dateien

Diese Seite hat keine eigenen Dateien. Die Konfigurationen stehen auf den verlinkten Themenseiten.
