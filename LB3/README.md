# LB3 Dokumntation
![image](https://github.com/yabber29/M300/blob/06c1c39667675c83fc304f3f21e6c73fef6d5a4d/Bilder/Openshift-Docker.png)
============================================================================================================

# Inahltsverzeichnis

 1. [Einführung](#Beschreibung) 
 2. [Projekt 1](#Projekt-Docker)
        1. [Netzwerkplan und Sicherheitskonzept](#sicherheit)
        2. [docker-compose.yaml](#yamlhehe)
        3. [Testing](#Testing)
 3. [Projekt 2](#Projekt-Openshift)
 4. [Reflexion](#Reflexion)
 5. [Wissenszuwachs](#Wissenszuwachs)
 6. [Quellen](#Quellen)

# Einführung

Ziel dieses Projekts ist es, mittels Docker-Compose eine funktionierende Umgebung zu erstellen. Alle nötigen Images sind im docker-compose.yaml-File enthalten und werden beim Ausführen des Commands automatisch geladen.

In diesem Projekt wird mittels docker-compose eine funktionierende Datenbank im Backend und Next-Cloud im Frontend verwendet. Der User kann dann unter localhost:80 Nextcloud einrichten.

# Netzwerkplan / Sicherheitskonzept

![image](https://github.com/yabber29/M300/blob/d93d921ed1fc07d3a5e98c987b40cba01233ed9f/Bilder/Netzwerkplan-Docker.png)

Wie hier oben im Plan zu sehen ist, sind zwei Ports auf dem Container offen:
| Port   |      Nutzen     |
|:----------|:-------------|
| 80 | Wird für den Zugriff auf das Nextcloud-GUI gebraucht. |
| 5432 | PostgreSQL-Port. Ist **NICHT** von aussen her erreichbar. Nur für Kommunikation unter Containern. |

Alle anderen Ports sind weder von aussen her, noch von intern her (Container untereindander) erreichbar. 


# docker-compose.yaml
In diesem Abschnitt erkläre ich, wie ich das YAML-File aufgebaut habe.

```yml
version: '3.2'
```
Die Version ist sehr wichtig. Nach mehrfachem Experimentieren und Abklären, ergab sich die Version 3.2 als die beste. Falls man die Version nicht ins File schreibt, gibt es Syntax-Fehlermeldungen beim Versuch, den Container mir docker-compose zu starten.

```yml
services:
```
Hier werden die Services definiert, ergo die Applikationen, die auf dem Container laufen werden. In unserem Fall wäre das einerseits der Service "Nextcloud" und andererseits der Service "".

```yml
 nc:
    image: nextcloud:apache
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_PASSWORD=Admin1234!!
      - POSTGRES_DB=hehedb
      - POSTGRES_USER=ramon
    ports:
      - 80:80
    restart: always
    volumes:
      - nc_data:/var/www/html

  db:
    image: postgres:alpine
    environment:
      - POSTGRES_PASSWORD=Admin1234!!
      - POSTGRES_DB=hehedb
      - POSTGRES_USER=ramon
    restart: always
    volumes:
      - db_data:/var/lib/postgresql/data
    expose:
      - 5432
```
Hier sehen wir unseren beiden Services. Was welche Zeile macht, erkläre ich hier:

| Befehl   |      Nutzen     |
|:----------|:-------------|
| nc: |Das ist die Beschreibung des Services "Nextcloud" (Abgekürzt auf nc). |
| image: |Dieses Image wird vom Docker-Hub (hub.docker.com) heruntergeladen. Nach erfolgreichem Herunterladen wird dieses Image extrahiert und lokal gestartet.|
| environment: | Dies sind Argumente, die man bei der Installation mitgeben kann. Beispielsweise sehen wir hier, dass der User ramon heissen wird, und das Passwort hirsch123 ist. |
| ports:  | Hier kann man definieren, welche Ports an den Container weitergeleitet werden. Es gilt --> **port_host:port_container**  |
| restart: | Beim Starten vom Docker-Deamon, wird dieser Container automatisch mitgestartet. |
| volumes:  | Volumes, die mitgegeben werden. |
| expose: | Beschreibt, welcher Port für die INTERNE Kommunikation freigegeben wird. Für eine genauere Beschreibung hilft das Kapitel Netzwerkplan / Sicherheitskonzept. |


# Testing

**Wichtig**: Der Service wird unter localhost:80 verfügbar sein, falls dieser Port bereits in Benutzung ist, bitte die Zeile **ports:** im YAML-File abändern (siehe Code unten): 

```yml
nc:
    image: nextcloud:apache
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_PASSWORD=Admin1234!!
      - POSTGRES_DB=hehedb
      - POSTGRES_USER=ramon
    ports:
      - <Beliebiger Port>:80  <-- Hier den Port ändern
    restart: always
    volumes:
      - nc_data:/var/www/html
```
Im Ordner, indem sich das YAML-File befindet, geben wir nun folgenden Command ein:

```yml
docker-compose up -d
```
Beim ersten Ausführen des Codes geht es ein wenig länger, da Docker zuerst die Images vom Dockerhub herunterladen muss. Anschliessend sollte folgendes zu sehen sein: 

![image]()

Anschliessend geben wir in einem beliebigen Browser die folgende URL ein: http://localhost:80 oder http://localhost:zuvor_festgelegter_Port

Man sollte nun folgendes sehen:
![image]()

Nun melden wir uns mit unseren Anmelde-Daten an, die im YAML-File zuvor definiert wurden:

| Username   |      Password     |
|:----------|:-------------|
| ramon | Admin1234!! |

Nun sollten wir eingeloggt sein. Das Dashboard sollte ungefähr so aussehen:
![image]()


Falls man den Container nicht mehr braucht, kann man ihn mit folgendem Command herunterfahren:

```yml
docker-compose down
```

Viel Spass beim Benutzen von Nextcloud :)

# Quellenangaben

| Objekt   |      Quelle     |
|:----------|:-------------|
| Docker-Compose generell | https://docs.docker.com/compose/gettingstarted/ |
| Gute Erklärung zu restart: always | https://serverfault.com/questions/884759/how-does-restart-always-policy-work-in-docker-compose |
