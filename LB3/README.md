# LB3 Dokumentation
![image](https://github.com/yabber29/M300/blob/06c1c39667675c83fc304f3f21e6c73fef6d5a4d/Bilder/Openshift-Docker.png)
============================================================================================================

# Inahltsverzeichnis

 1. [Beschreibung](#Einführung) 
 2. [Projekt-Docker](#Projekt-1)
    1. [Netzwerkplan und Sicherheitskonzept](#Netzwerk)
    2. [docker-compose.yaml](#Dockerfile)
    3. [Testing](#Testing)
 3. [Projekt-Openshift](#Projekt-2)
    1. [Netzwerkplan und Sicherheitskonzept](#Netzwerk-Overview)
    2. [Dokumentenstruktur](#Struktur)
    3. [Testing](#Testing-Files)
 4. [Reflexion](#Reflexion)
 5. [Wissenszuwachs](#Wissenszuwachs)
 6. [Quellen](#Quellen)

# Einführung

Ziel dieses Projekts ist es, mittels Docker-Compose eine funktionierende Umgebung zu erstellen. Alle nötigen Images sind im docker-compose.yaml-File enthalten und werden beim Ausführen des Commands automatisch geladen.

In diesem Projekt wird mittels docker-compose eine funktionierende Datenbank im Backend und Next-Cloud im Frontend verwendet. Der User kann dann unter localhost:80 Nextcloud einrichten.

# Projekt-1

## Netzwerk

![image](https://github.com/yabber29/M300/blob/d93d921ed1fc07d3a5e98c987b40cba01233ed9f/Bilder/Netzwerkplan-Docker.png)

Wie hier oben im Plan zu sehen ist, sind zwei Ports auf dem Container offen:
| Port   |      Nutzen     |
|:----------|:-------------|
| 80 | Wird für den Zugriff auf das Nextcloud-GUI gebraucht. |
| 5432 | PostgreSQL-Port. Ist **NICHT** von aussen her erreichbar. Nur für Kommunikation unter Containern. |

Alle anderen Ports sind weder von aussen her, noch von intern her (Container untereindander) erreichbar. 


## Dockerfile
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
| environment: | Dies sind Argumente, die man bei der Installation mitgeben kann. Beispielsweise sehen wir hier, dass der User ramon heissen wird, und das Passwort Admin1234!! ist. |
| ports:  | Hier kann man definieren, welche Ports an den Container weitergeleitet werden. Es gilt --> **port_host:port_container**  |
| restart: | Beim Starten vom Docker-Deamon, wird dieser Container automatisch mitgestartet. |
| volumes:  | Volumes, die mitgegeben werden. |
| expose: | Beschreibt, welcher Port für die INTERNE Kommunikation freigegeben wird. Für eine genauere Beschreibung hilft das Kapitel Netzwerkplan / Sicherheitskonzept. |


## Testing

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

![image](https://github.com/yabber29/M300/blob/a4f2dc9726fb99e724777dc582aaf2bdb768d597/Bilder/docker-compose.png)

Anschliessend geben wir in einem beliebigen Browser die folgende URL ein: http://localhost:80 oder http://localhost:zuvor_festgelegter_Port

Man sollte nun folgendes sehen:
![image](https://github.com/yabber29/M300/blob/a4f2dc9726fb99e724777dc582aaf2bdb768d597/Bilder/login-nextcloud.png)

Nun melden wir uns mit unseren Anmelde-Daten an, die im YAML-File zuvor definiert wurden:

| Username   |      Password     |
|:----------|:-------------|
| ramon | Admin1234!! |

Nun sollten wir eingeloggt sein. Das Dashboard sollte ungefähr so aussehen:
![image](https://github.com/yabber29/M300/blob/a4f2dc9726fb99e724777dc582aaf2bdb768d597/Bilder/nextcloud.png)


Falls man den Container nicht mehr braucht, kann man ihn mit folgendem Command herunterfahren:

```yml
docker-compose down
```

Viel Spass beim Nachmachen von Nextcloud :)

# Projekt-2
In unserem Projekt automatisieren wir das Deployment und die Aktualisierung einer Webanwendung, die aus einer Index.html und weiteren Seiten besteht. Wir verwenden BitBucket für die Versionsverwaltung und ArgoCD für die Verbindung und kontinuierliche Bereitstellung der Anwendung. Die Infrastruktur läuft auf OpenShift, einer Container-Plattform basierend auf Kubernetes.
Wir speichern die Dateien der Anwendung lokal, und die Bereitstellung erfolgt mithilfe eines Helm-Charts, das im Templates-Ordner liegt. Der Helm-Chart ermöglicht uns die einfache Verwaltung und Aktualisierung von Kubernetes-Ressourcen, wie zum Beispiel der Route für den Zugriff auf die Anwendung.
Durch diese automatisierte Pipeline können wir schnell Änderungen vornehmen und bereitstellen, wodurch eine effiziente und agile Entwicklungsumgebung geschaffen wird.


## Netzwerk-Overview
Damit ich mir das ganze besser vorstellen konnte erstellte ich zuerst ein Netzwerkplan.
![image](https://github.com/yabber29/M300/blob/7eb6d4cfea5e556d1bfed118ae8bb7b2b10ce457/Bilder/Netzwerkplan-Openshift.png)

## Struktur
Die einzelne Prozzese werden hier dokumentiert:

### BitBucket
BitBucket ist ein webbasierter Dienst für Versionsverwaltung, der Git und Mercurial unterstützt. Entwickelt von Atlassian, ermöglicht BitBucket Teams, Quellcode in Repositories zu speichern, zu verwalten und gemeinsam daran zu arbeiten. BitBucket bietet Funktionen wie Pull-Anfragen, Issue-Tracking, Continuous Integration und Deployment Pipelines. Es ist speziell für die Zusammenarbeit in Teams konzipiert und hilft, den Softwareentwicklungsprozess zu optimieren und die Produktivität zu steigern.

### Openshift
OpenShift ist eine Container-Plattform, die von Red Hat entwickelt wurde und auf Kubernetes basiert. Es erleichtert das Erstellen, Bereitstellen und Verwalten von containerisierten Anwendungen und Microservices in einer Cloud-Umgebung. OpenShift bietet Entwicklern eine benutzerfreundliche Umgebung, automatisiert die Verwaltung von Containern und unterstützt verschiedene Programmiersprachen, Datenbanken und Frameworks. Es hilft Unternehmen dabei, ihre Entwicklungs- und Betriebsabläufe zu optimieren und eine effiziente, skalierbare und sichere Infrastruktur aufzubauen.

### Projekt in Openshift
Ein Projekt in OpenShift ist eine logische Einheit, die mehrere Anwendungen, Ressourcen und Benutzer zusammenfasst. Projekte helfen dabei, die Ressourcen in einem OpenShift-Cluster zu organisieren und zu isolieren. Sie sind das OpenShift-Äquivalent zu Kubernetes-Namensräumen und bieten zusätzliche Funktionen zur Verwaltung von Zugriffsrechten, Ressourcenquoten und -grenzen.



### Deployment mit Webserver
Ein Deployment ist eine Kubernetes-Ressource, die den gewünschten Zustand einer Anwendung definiert und deren Skalierung, Aktualisierung und Wiederherstellung automatisiert. Bei einem Webserver-Deployment wird eine Webserver-Anwendung, wie z.B. Apache oder Nginx, in einem oder mehreren Containern bereitgestellt.
Das Deployment stellt sicher, dass eine bestimmte Anzahl von Replikaten des Webservers immer ausgeführt wird und aktualisiert die Container automatisch bei Änderungen an der Konfiguration oder dem Container-Image. Dadurch wird die Verfügbarkeit und Skalierbarkeit der Webanwendung gewährleistet.

![image](https://github.com/SilvioM04/M300/blob/859ff7d6c424827c2ad39a8d09cbdbb8894f915c/Bilder/httpd3.png)
![image](https://github.com/SilvioM04/M300/blob/859ff7d6c424827c2ad39a8d09cbdbb8894f915c/Bilder/httpd1.png)
![image](https://github.com/SilvioM04/M300/blob/859ff7d6c424827c2ad39a8d09cbdbb8894f915c/Bilder/httpd2.png)

### ArgoCD

#### Theorie

#### Praxis

### Helm

#### Theorie

#### Praxis
### ConfigMaps
#### Theorie
#### Praxis

## Testing-Files

# Reflexion

# Wissenszuwachs

# Quellenangaben

