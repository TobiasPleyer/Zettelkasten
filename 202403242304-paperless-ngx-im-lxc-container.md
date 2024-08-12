---
date:  Sunday, March 24, 2024
tags:
---

# Paperless-ngx im LXC Container

[Original](https://www.schreiners-it.de/proxmox/paperless-ngx-im-lxc-container-installieren/)

Zum Inhalt springen
Schreiners IT
Schreiners IT

Gedankenarchiv eines Geek's

Hauptmenü

  • NewsMenü umschalten
      □ Updates
  • CheckMK
  • Nextcloud
  • Nginx Proxy Manager
  • Pi Hole
  • Proxmox
  • Raspberry Pi
  • Synology
  • Vaultwarden
  • Über mich
  • Kontakt
  •  
    Spenden
    Spenden

8. April 2022 / Proxmox / Von Lucas Schreiner

Paperless-ngx im LXC Container installieren

[paperless-ngx-logo]

Moin!

Vielleicht hat schon mal jemand von euch etwas von Paperless bzw. Paperless-ng
oder Paperless-ngx gehört.
Paperless-ngx ist ein Tool, um sein Büro und die damit verbundene
Papierwirtschaft zu digitalisieren. Es ist möglich, Dokumente mit OCR zu
scannen und anschließend mit Tags zu versehen.

Des Weiteren können Korrespondenten erstellt werden, denen Schlagwörter
zugeordnet werden. So kann Paperless-ngx automatisch neue Dokumente ordnen.
Ein weiteres Feature ist, dass Paperless ein sogenanntes consume Verzeichnis
hat. So kann z.B. mit einem Scanner ein Dokument eingescannt werden und per FTP
oder SMB/CIFS automatisiert in dem consume Verzeichnis abgelegt werden. Das
Besondere an dieser Installation gegenüber vieler anderen ist, dass ich keine
Volumen nutze. Wir mounten die benötigten Ordner direkt auf den Docker Host.
Dies vereinfacht das Backup der Daten aus dem Host heraus enorm.

Weiter unten findet ihr noch zusätzliche Einstellungen, die optional sind und
nicht für den fehlerfreien Betrieb von paperless nötig sind. Solltet ihr
Paperless bereits installiert haben und lediglich die zusätzlichen
Einstellungen übernehmen wollen, gleicht bitte eure Installation mit meiner ab,
um potenzielle Fehler oder Inkompatibilitäten zu vermeiden. Generell werde ich
nur Lesern helfen können, die die Installation nach meinem Beitrag getätigt
haben. Der Zeitaufwand dafür ist einfach zu hoch, um dies kostenfrei anbieten
zu können.

Nach der erfolgreichen Installation widmen wir uns der Frage, wie Paperless bei
dem Erscheinen einer neuen Version aktualisiert werden kann. Dies gestaltet
sich sehr einfach und ist schnell erledigt, wenn man etwas Routine hat, welche
aber zügig kommt.

Noch weiter unten findet ihr hingegen Einstellungen, die bei der Beseitigung
von Problemen helfen können, aber nicht generell, für alle Anwender gesetzt
werden müssen. Diese sollten auch nur gesetzt werden, wenn ihr das
entsprechende Problem auf eurem System vorfindet.

WICHTIG: Ich werde hier nur freiwilligen Support leisten, WENN Docker und
Docker compose nach meiner Anleitung installiert wurden, dies sorgt dafür, dass
einige Fehler gar nicht erst entstehen. Zudem erleichtert es enorm die Arbeit,
da ich keine falschen oder mangelhaften Installationen korrigieren muss.
Zusätzlich supporte ich nur aktuelle Version einer Docker Installationen, um
potenzielle Fehler auszuschließen.

Wir benötigen dazu Folgendes:

 1. LXC Container mit Docker, Docker-compose und Portainer
 2. 8 GB Speicherplatz im LXC
 3. Zweite Disk mit 10 GB Speicherkapazität
 4. Scanner mit SMB/CIFS

Für die Installation verwenden wir 2 Disks im LXC Container. Das hat den
Vorteil, dass das die Daten und ein potenzielles Backup schnell getrennt und
erneut eingespielt werden können. Dazu ruft Ihr im Proxmox Web UI einfach den
Container auf und wählt oben links „Hinzufügen“ aus. Nun könnt ihr eine zweite
virtuelle Harddisk hinzufügen. Diese müsst ihr unter „/data“ mounten.

Wir verbinden uns per SSH auf den LXC Container und stoßen ein Update & Upgrade
an.
Danach installieren wir Samba, welches wir für die Netzwerkfreigabe benötigen.

sudo apt update && sudo apt upgrade -y && sudo apt install samba -y

Damit später auch Umlaute korrekt auf der Kommandozeile angezeigt werden,
prüfen wir die „locales“ Einstellungen unseres Hosts. Wer den LXC Container
nach meiner Anleitung konfiguriert hat, kann diesen Schritt getrost
überspringen. Dazu feuern wir die folgende Befehle ab und setzen so die
korrekte „locales“ Einstelleung für den deutschen Sprachraum.

echo "de_DE.UTF-8 UTF-8" > /etc/locale.gen
dpkg-reconfigure --frontend=noninteractive locales

echo -e "LANG=de_DE.UTF-8\nLANGUAGE=\"de_DE:de\"" >> /etc/default/locale

Um nun die neue „locale“ Einstellung nutzen zu können müssen wir entweder einen
Neustart des LXCs vornehmen oder den aktuellen User einfach ausloggen und
erneut einloggen. Ich habe mich in diesem Beitrag für das letztere entschieden.

logout

Anschließend erstellen wir einige Verzeichnisse, welche für die Installation
benötigt werden. Der Ordner „consume“ wird im weiter laufenden Beitrag per SMB
im Netzwerk verteilt.

cd /
sudo mkdir data
sudo mkdir data/paperless
sudo mkdir data/paperless/consume
sudo mkdir data/paperless/data
sudo mkdir data/paperless/export
sudo mkdir data/paperless/media
sudo mkdir data/paperless/redis
sudo mkdir data/paperless/postgresql

Wir legen einen Nutzer für die Samba-Freigabe an, diesen Nutzer verwenden wir
auch später für Paperless. Den erstellten User fügen wir der Docker Gruppe
hinzu, damit dieser Container verwalten kann. Anschließend erstellen wir ein
SMB Passwort, damit der User in der Lage ist sich gegen den SMB Service zu
authentifizieren.

sudo adduser paperless
sudo usermod -aG docker paperless
sudo smbpasswd -a paperless

Ist dies geschafft, legen wir ein Backup der smb.conf an. Danach passen wir
unsere „smb.conf“ an. Dazu editieren wir einige bereits bestehende Werte und
fügen neue Parameter hinzu.
Leider haben Leser vereinzelt Probleme, die Config korrekt zu übernehmen, daher
habe ich diese nun stark vereinfacht. Ihr müsst einfach alles unterhalb „Share
Definitions“ entfernen und durch die Parameter in der Codebox ersetzen.

sudo nano /etc/samba/smb.conf

#======================= Share Definitions =======================

[global]
map to guest = never

[homes]
comment = Home Directories
browsable = no
read only = yes
valid users = %S
create mask = 0700
directory mask = 0700

[paperless]
comment = Paperless SMB Consume
valid users = paperless
path = /data/paperless/consume/
public = no
writable = yes
printable = no
browsable = no
guest ok = no
create mask = 0700
directory mask = 0700

Jetzt müssen wir die Berechtigungen auf den Samba Ordner anpassen, damit unser
User auch Zugriff hat. Anschließend starten wir den smb Dienst neu.

sudo chown -R paperless:paperless /data/paperless
sudo chmod 700 /data/paperless
sudo systemctl restart smbd

Nun wechseln wir zu unserem User „paperless“. Anschließend springen wir in das
Userverzeichnis von „paperless“ und legen die Konfigurationsdatei für unsere
Paperless Applikation an. Für die User zu Ordnung sollte zudem die „UID“ und
„GID“ des „paperless“ Users ermittelt werden. Dies sind auf eurem System
einzigartige IDs, die nur dem jeweiligen User zu geordnet sind. Dafür fragen
wir den Inhalt der „group“ Datei ab.

su paperless
cd /home/paperless
cat /etc/group | grep paperless:x:

Die nun ermittelte ID entspricht der „UID“, sowie der „GID“. Diese tragen wir
in der unten stehende Config unter den entsprechenden Parametern ein. Zudem
muss in der „docker-compose.yml“ unter „webserver“ noch den Admin User und das
entsprechende Passwort definieren. Ich empfehle „paperless“ als User
eingetragen zu lassen und entsprechend euer Wunschpasswort für den Admin User
festzulegen.

nano docker-compose.yml

version: "3.4"
services:
  broker:
    image: docker.io/library/redis:7
    container_name: broker
    restart: unless-stopped
    volumes:
      - /data/paperless/redis/_data:/data

  db:
    image: docker.io/library/postgres:13
    container_name: db
    restart: unless-stopped
    volumes:
       - /data/paperless/postgresql/_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: paperless
      POSTGRES_USER: paperless
      POSTGRES_PASSWORD: paperless

  webserver:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    container_name: webserver
    restart: unless-stopped
    depends_on:
      - db
      - broker
      - gotenberg
      - tika
    ports:
      - 8001:8000
    healthcheck:
      test: ["CMD", "curl", "-fs", "-S", "--max-time", "2", "http://localhost:8000"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      - /data/paperless/consume:/usr/src/paperless/consume
      - /data/paperless/data:/usr/src/paperless/data
      - /data/paperless/media:/usr/src/paperless/media
      - /data/paperless/export:/usr/src/paperless/export
    environment:
      PAPERLESS_ADMIN_USER: paperless
      PAPERLESS_ADMIN_PASSWORD: EuerAdminPasswortEintragen
      PAPERLESS_REDIS: redis://broker:6379
      PAPERLESS_DBHOST: db
      PAPERLESS_TIKA_ENABLED: 1
      PAPERLESS_TIKA_GOTENBERG_ENDPOINT: http://gotenberg:3000
      PAPERLESS_TIKA_ENDPOINT: http://tika:9998
      PAPERLESS_OCR_LANGUAGE: deu
      PAPERLESS_TIME_ZONE: Europe/Berlin
      USERMAP_UID: IDdesPaperlessUsers
      USERMAP_GID: IDdesPaperlessUsers

  gotenberg:
    image: docker.io/gotenberg/gotenberg:latest
    container_name: gotenberg
    restart: unless-stopped
    command:
      - "gotenberg"
      - "--chromium-disable-routes=true"
      - "--chromium-allow-list=file:///tmp/.*"

  tika:
    image: ghcr.io/paperless-ngx/tika:latest
    container_name: tika
    restart: unless-stopped

Nun können wir unsere Container starten. Dies geschieht ganz einfach über ein
wohlbekanntes Kommando.

docker compose up -d

Ihr könnt abschließend das Web UI von Paperless über die xxx.xxx.xxx.xxx:8001
erreichen.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Zusätzliche Einstellungen für Paperless

Nachfolgend werde ich einige für euch eventuell sinnvolle Einstellungen
präsentieren. Dabei kann jeder für sich selbst überlegen, ob er diese
übernehmen möchte oder nicht. Keine dieser Einstellungen ist für den
fehlerfreien Betrieb von Paperless nötig.

  • Paperless über Domain aufrufen
  • Backup erstellen und konfigurieren
  • Daten aus Backup wiederherstellen
  • Netzwerklaufwerk unter Windoof einbinden
  • Ordner- und Dateinamenstruktur auf eigene Bedürfnisse anpassen
  • Dateien nach Änderung der Dateinamenstruktur automatisch umbenennen

Paperless über Domain aufrufen

Es ist möglich Paperless intern über den Hostname in Verbindung mit der lokalen
Domain aufzurufen. Dies ist zudem auch extern über den Hostname in Verbindung
mit Sub-/Domain möglich. Vorab gilt aber zu beachten, dass die Entwickler stark
davon abraten, die Installation EXTERN erreichbar zu machen! Dennoch haben
Leser bereits danach gefragt.
Voraussetzung dafür ist korrekt konfigurierter DNS-Server sowie eine lokale
Domain. Auch eine normale TLD, die sich in eurem Besitz befindet, kann dafür
lokal genutzt werden. Ich empfehle dies eher Lesern, die schon etwas mehr
Grundwissen besitzen.

Solltet Ihr Paperless über den Hostname bzw. in Verbindung mit der lokalen
Domain aufrufen wollen, müssen wir einen Parameter unter dem Punkt
„environment“ hinzufügen.
Achtet dabei darauf, dass ihr entweder die korrekte lokale Domain oder eine
normale TLD, die euch gehört, dafür verwendet. Kein Mischmasch. Wir zerstören
zunächst den Container und fügen danach den Parameter der „docker-compose.yml“
hinzu.

cd /home/paperless/
docker compose down

nano docker-compose.yml

    environment:
      PAPERLESS_URL: http://paperless.localdomain.tld

Ist das geschafft bauen wir uns wieder einen Paperless Container

docker compose up -d

Backup erstellen und konfigurieren

Abschließend richten wir das Backup noch ein. Dazu verwenden wir ein Paperless
eigenes Tool, welches sich „document_exporter“ schimpft. Dann basteln wir uns
noch ein kleines Shell Kommando zusammen, was ein Backup in unseren „export“
Ordner anlegt. In welchem Zyklus ihr das Backup anlegen wollt, ist euch
überlassen. Ich habe jeden Tag um 1 Uhr morgens gewählt, da es sich
schlussendlich um relevanten Daten handelt.

crontab -e

0 1 * * * docker compose -f /home/paperless/docker-compose.yml exec -T webserver document_exporter /usr/src/paperless/export/

Um den Shellbefehl verstehen zu können, schlüssel ich diesen kurz auf.

docker compose -f /home/paperless/docker-compose.yml exec -T webserver document_exporter /usr/src/paperless/export/

Der erste Teil bis „exec“ sorgt dafür, dass ein „docker compose“ Kommando
außerhalb des „paperless“ Ordner ausgeführt werden kann. Von „exec“ bis
einschließlich „webserver“ ermöglicht das Ausführen eines Kommandos innerhalb
des „webserver“ Containers. „document_exporter“ ist das Kommando, welches das
Tool aufruft. Der letzte Teil ist der in den „webserver“ Container gemountete
Pfad des Export-Ordners.

Daten aus Backup wiederherstellen

Das Wiederherstellen von Daten aus einem Backup gestaltet sich relativ einfach,
da es wie bei dem Backup auch hier ein integriertes Tool gibt, welches wir
verwenden können. Das besagte Tool nennt sich in diesen Fall
„document_importer“. Wie bei dem Backup geben wir auch hier wieder den Pfad des
„Export“ Verzeichnisses an, welches wir direkt in dem Container „webserver“
ausführen. Wichtig hierbei ist, dass wir genauso wie bei dem Backup, den in den
Container gemounteten „Export“ Pfad auswählen.

docker compose -f /home/paperless/docker-compose.yml exec -T webserver document_importer /usr/src/paperless/export/

Netzwerklaufwerk unter Windoof einbinden

Nun könnt ihr unter Windows das Netzwerklaufwerk einbinden. Dies ermöglicht
euch das automatische Konsumieren neuer Dokumente. Dazu müssen diese einfach in
das Laufwerk gelegt werden. Dazu öffnet ihr einfach „Dieser PC“ oder den
„Explorer“. Dort findet ihr im oberen Bereich den Punkt „Netzwerklaufwerk
verbinden“. Danach tragt ihr folgenden Parameter ein und startet den
Verbindungsvorgang. Jetzt noch die Logindaten eintragen und der Drops ist
gelutscht.

\\IP.von.Paperless.hier\paperless\

Ordner- und Dateinamensstruktur auf eigene Bedürfnisse anpassen

Paperless gestattet es einem, die Ordner- und Dateinamensstruktur anzupassen.
Dies ermöglicht eine Art von Ordnerablage, was der Organisation vieler Leute
entspricht, die schon vor Paperless ihre Dokumente digital verwaltet haben. So
ist es z.B. möglich, übergeordnete Ordner geordnet nach Jahren und
untergeordnete Ordner für Korrespondenten anzulegen. Dies ist möglich, indem
verschiedene Platzhalter aneinander gereiht werden. Eine Übersicht der
möglichen Platzhalter findet ihr in der offiziellen Paperless Dokumentation.
Soll zwischen Platzhalter eine Ordnerebene hinzugefügt werden, geschieht dies
durch „/“ Slashes. Einzelne Platzhalter können innerhalb einer Ordnerebene
durch Zeichen wie Bindestriche, Unterstriche und Leerzeichen voneinander
getrennt werden.

Solltet ihr bereits Dokumente in Paperless eingepflegt haben, muss der
nachfolgende Schritt ebenfalls umgesetzt werden. Sonst wird die neue Datei-/
und Ordnernamensstruktur lediglich für neu eingepflegte Dokumente angewendet
und nicht für bereits bestehendes Material.

Zur Veranschaulichung hier ein kleines theoretisches Beispiel:

  • 2023
      □ Korrespondenten
          ☆ Dokumententitel
  • 2022
      □ Korrespondenten
          ☆ Dokumententitel

Im praktischen Umfeld kann dies dann wie folgt aussehen:

  • 2023
      □ Meine Bank
          ☆ 02-01-2023 Vertragsabschluss
          ☆ 02-01-2023 Vertragsbestätigung
      □ KFZ Versicherung
          ☆ 01-01-2022 Halbjahresrechnung

  • 2022
      □ KFZ Versicherung
          ☆ 01-01-2022 Halbjahresrechnung
          ☆ 01-07-2022 Halbjahresrechnung

Um nun das oben gezeigte umsetzen zu können, müssen wir zunächst den Container
wieder zerstören.

cd /home/paperless/
docker compose down

Jetzt können wir der „docker-compose.yml“ den erforderlichen Parameter
hinzufügen. Für das oben gezeigte Beispiel wäre diese Konfiguration nötig. Die
Zeile kann nach eurem Belieben abgeändert werden. Achtet nur auf die
Begrenzungen seitens Paperless.

nano docker-compose.yml

    environment:
      PAPERLESS_FILENAME_FORMAT: "{created_year}/{correspondent}/{created_day}-{created_month}-{created_year} {title}"

Dateien und Ordner nach Änderung der Datei- und Ordnernamensstruktur
automatisch umbenennen

Solltet ihr bereits Dokumente in Paperless abgelegt haben, bevor ihr die
Dateinamensstruktur angepasst habt, müssen alle schon existierenden Dokumente
neu benannt werden, damit sie dem Schema, welches ihr vorher definiert habt,
entsprechen. Klingt nach Spaß, oder? 😛 Um uns viel Mühen zu ersparen, haben die
Entwickler rund um Paperless auch für diesen Fall ein kleines Tool
bereitgestellt. Es handelt sich dabei um den „document_renamer“, welcher bei
Aufruf die vorhandenen Dateien nach dem definierten Schema umbenennt. Die
Bedienung des Tools ist unfassbar einfach. Wir müssen es lediglich im Paperless
Container ausführen.

docker compose -f /home/paperless/docker-compose.yml exec -T webserver document_renamer

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Update von Paperless-ngx

Das Aktualisieren von Paperless gestaltet sich relativ einfach, da wir es
Docker basierend installiert haben. Zunächst springen wir in das Verzeichnis,
in der die „docker-compose.yml“ liegt. Zur vorsorglichen Datensicherung stoßen
wir ein Backup an, wie dies manuell und/oder automatisch ablaufen sollte, könnt
ihr weiter oben nachlesen, daher gehe ich hier nicht weiter darauf ein.
Anschließend schalten wir den Container ab, ziehen uns das aktuellste Docker
Image aus dem Repository und starten den Container auf Basis des neuen Images.
Die Daten wie Dokumente und Datenbank blieb erhalten, da diese direkt auf
Verzeichnisse des Docker Hosts gemountet sind.

cd /home/paperless/
docker compose down
docker compose pull
docker compose up -d

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Einstellungen zur Beseitigung von Problemen

Es gibt die ein oder andere Einstellung, die optional gesetzt werden kann, WENN
es auf dem betreffenden System zu einer Störung oder einem Problem kommen
sollte. Ich werde entsprechend ein Problem schildern und dazu eine mögliche
Lösung eintragen. Dies kann das Problem lösen oder auch nicht. Häufig werden
diese Troubleshooting Ansätze von meinen Lesern kommen, ich liste diese hier
auf, eine gewisse Übersicht anzubieten.

  • Paperless hat Vollzugriff auf das consume Verzeichnis, scannt aber keine
    Dokumente
  • failed to register layer: ApplyLayer exit status 1 stdout:

Paperless hat Vollzugriff auf das Verzeichnis, scannt aber keine Dokumente

Vereinzelt haben Nutzer berichtet, dass Paperless keine Dokumente aus dem
consume Ordner lesen kann. Zunächst sollte geprüft werden, ob der Nutzer
„paperless“ ausreichende Berechtigung hat und zudem noch Besitzer der
Ordnerstruktur ist. Nachdem dies geprüft wurde, kann es helfen Paperless
manuell anzuweisen in einem gewissen zeitlichen Zyklus das consume Verzeichnis
scannen zu lassen. Im Normalfall erkennt Paperless durch Änderungen des
Filesystems, dass neue Dateien im Verzeichnis liegen.
Dazu fügen wir einfach der „docker-compose.yml“ einen Parameter hinzu. Die Zahl
hinter dem Parameter gibt die Zykluszeit in Sekunden an.

cd /home/paperless/
docker compose down

nano docker-compose.yml

    environment:
      PAPERLESS_CONSUMER_POLLING: 60

docker compose up -d

failed to register layer: ApplyLayer exit status 1

failed to register layer: ApplyLayer exit status 1 stdout: stderr: unlinkat /var/cache/apt/archives: invalid argument

Scheinbar gibt es seit einiger Zeit ein Problem, welches auftritt, wenn Docker
in einem Proxmox LXC Container auf einem ZFS Storage betrieben wird. Es sind
ledig Nutzer von ZFS betroffen und scheinbar bei Weitem nicht alle. Keins der
von mir betreuten System, welche aktuell ZFS verwende, hatte bis jetzt dieses
Problem. Zum jetzigen Zeitpunkt scheint es so, als könnte der aktuelle Opt-In
Kernel 6.1 von Proxmox dort Abhilfe schaffen, zumindest berichten dies
vereinzelt User im Proxmox Forum.

Das Problem tritt auf, wenn der Storage Driver ZFS verwendet wird. In der
Vergangenheit wurde an der Stelle automatisch der VFS Storage Driver
angewendet, dieser bläht allerdings das Dateisystem auf und erhöht damit den
einhergehenden Speicherplatzverbrauch. Leider gibt es zum aktuellen Zeitpunkt
keine andere für Laien anwendbare Lösung als auf den besagten VFS Driver zu
wechseln. Es besteht die Möglichkeit „fuse“ zu nutzen, davon rate ich aber
Dringen ab! Sehr dringend! Finger weg!

Dazu müssen wir zunächst den Docker Service stoppen und anschließend die
„daemon.json“ anpassen. Dort tragen wir ein, dass der „VFS“ Driver verwendet
werden soll.

sudo systemctl stop docker.service

nano /etc/docker/daemon.json

{
  "storage-driver": "vfs"
}

Nun treten wir den Docker Service einmal durch.

sudo systemctl start docker.service

Abschließend prüfen wir, ob Docker die Änderung angenommen und den Storage
Driver korrigiert wurde. Dazu setzen wir ein kleines Kommando ab und prüfen die
Ausgabe. Diese sollte klar den Storage Driver „VFS“ anzeigen.

docker info | grep Storage

Storage Driver: vfs

Grüße gehen aus dem Archiv!

Post navigation
← Vorheriger Beitrag
Nächster Beitrag →
Abonnieren
Anmelden
Benachrichtige mich bei
[allen neuen Kommentare              ]
[                    ]
[›]
[ ] Ich erlaube meine Email Adresse dazu zu nutzen um mich über neue Kommentare
und Antworten zu benachrichtigen. (Du kannst jederzeit der Zustimmung
wiedersprechen.)
Label [                    ]
[                    ] Name*
[                    ] E-Mail*
[                    ] Bitte die "Glocke" aktivieren, damit eine Benachritigung
stattfindet, wenn auf den Kommentar geantwortet wird.
[ ] [Kommentar absenden]
Label [                    ]
[                    ] Name*
[                    ] E-Mail*
[                    ] Bitte die "Glocke" aktivieren, damit eine Benachritigung
stattfindet, wenn auf den Kommentar geantwortet wird.
[ ] [Kommentar absenden]
118 Kommentare
Älteste
Neuste Meist Bewerteste
Inline Feedbacks
Zeige alle Kommentare
Weitere Kommentare anzeigen
Suchen nach: [                    ] [Suche]
Kategorien

  • CheckMK (6)
  • Minecraft (2)
  • News (3)
  • Nextcloud (3)
  • Nginx Proxy Manager (4)
  • Pi Hole (3)
  • Proxmox (15)
  • Raspberry Pi (6)
  • Synology (4)
  • Vaultwarden (4)

Neueste Beiträge

  • Proxmox „No Subscription…“ Meldung entfernen 7. Oktober 2022
  • Neue Domain und Umbennenung des Archivs! 29. August 2022
  • Die Fritz!Box mit CheckMK überwachen 17. Juni 2022
  • Pushover in CheckMK einrichten 10. Juni 2022
  • Backup und Restore bei CheckMK 3. Juni 2022
  • CheckMK auf neue Version updaten 25. Mai 2022
  • CheckMK im LXC Container unter Pimox v7 20. Mai 2022
  • CheckMK im LXC Container installieren 14. Mai 2022
  • Vielen Dank für eure Hilfe! 29. April 2022
  • JDownloader 2 auf Debian 11 installieren 22. April 2022

  • Oktober 2022
  • August 2022
  • Juni 2022
  • Mai 2022
  • April 2022
  • März 2022
  • Februar 2022
  • Januar 2022
  • Dezember 2021
  • November 2021
  • August 2021
  • Mai 2021
  • März 2021
  • Dezember 2020
  • November 2020
  • Oktober 2020

  • Impressum
  • Datenschutzerklärung
  • Privatsphäre-Einstellungen ändern
  • Historie der Privatsphäre-Einstellungen
  • Einwilligungen widerrufen

Copyright © 2020 - 2024 Schreiners IT | Lucas Schreiner
wpDiscuz
118
0
Bitte lasse uns an deinen Gedanken teilhaben und kommentier den Beitrag.x
()
x
| Antworten
[                    ] Insert
Cookie Consent mit Real Cookie Banner 
