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

Hier werden ganz einfach den Namen, welches Projekt, den Port und welche Applikation definiert.

![image](https://github.com/SilvioM04/M300/blob/859ff7d6c424827c2ad39a8d09cbdbb8894f915c/Bilder/httpd1.png)

In spec/volumes werden die ConfigMaps so benannt wie sie als Datei in OpenShift gespeichert werden, damit die Websiten funktionieren.


![image](https://github.com/SilvioM04/M300/blob/859ff7d6c424827c2ad39a8d09cbdbb8894f915c/Bilder/httpd2.png)

Und unter containers/volumeMounts wird angegeben unter welchem Ordner diese gespeichert werden.

### ArgoCD
#### Theorie
Argo CD ist ein benutzerfreundliches Open-Source-Tool für das kontinuierliche Bereitstellen von Anwendungen in Kubernetes-Umgebungen. Es basiert auf den GitOps-Prinzipien und macht das Verwalten von Konfigurationen kinderleicht. Mit Argo CD können Anwendungen automatisch aus Git-Repositories bereitgestellt werden, wobei stets darauf geachtet wird, dass der gewünschte Zustand der Anwendung mit dem aktuellen übereinstimmt. Zudem profitieren Benutzer von hilfreichen Funktionen wie Versionskontrolle, der Möglichkeit, zu früheren Versionen zurückzukehren, und Überwachungstools, die den Bereitstellungsprozess effizienter, sicherer und verlässlicher gestalten.

#### Praxis
Hier wäre ein Beispiel, wie man es machen könnte, wir haben uns an diesem Beispiel orientiert.

1. **Dateien ins BitBucket-Repository übertragen**: Da wir bereits ein BitBucket-Repository haben, übertragen wir unsere Dateien dorthin, einschließlich der Helm-Chart-Struktur und der ConfigMap, die unsere index.html, CSS- und Python-Dateien enthält.
2. **ArgoCD-Projekt erstellen**: Wir melden uns bei der ArgoCD-Webkonsole an, die bereits in unserem OpenShift-Cluster installiert ist. Wir erstellen ein neues Projekt, um unsere Anwendung zu verwalten. Wir geben unserem Projekt einen Namen, z. B. "unsere-webapp-projekt", und konfigurieren die Zugriffsrechte und Clusterressourcen entsprechend unseren Anforderungen.
3. **ArgoCD-Anwendung erstellen**: Wir erstellen innerhalb des Projekts eine neue ArgoCD-Anwendung. Wir geben die erforderlichen Details an, wie:
  * Anwendungsname: z.B. "unsere-webapp"
  * Projekt: Wir wählen das zuvor erstellte Projekt "unsere-webapp-projekt"
  * Synchronisationsrichtlinie: Automatisch oder manuell, je nach unseren Präferenzen
  * Repository-URL: Die URL unseres BitBucket-Repositorys
  * Revisionsverlauf: Der Branch oder Tag, den wir verfolgen möchten (z. B. "master" oder "main")
  * Pfad: Wir geben den Pfad zu unserer Helm-Chart an, z. B. "unsere-webapp"
  * Cluster-URL: Die URL unseres OpenShift-Clusters
  * Namespace: Der OpenShift-Projektname, in dem wir die Anwendung bereitstellen möchten
4. **ConfigMap hinzufügen**: Um unsere index.html, CSS- und Python-Dateien in der Anwendung bereitzustellen, erstellen wir eine ConfigMap und speichern die Dateien darin. In unserer Helm-Chart fügen wir die ConfigMap in den templates-Ordner ein und binden diese ConfigMap in der deployment.yaml-Datei ein, damit unsere Webanwendung auf die in der ConfigMap gespeicherten Dateien zugreifen kann.

![image](https://github.com/SilvioM04/M300/blob/859ff7d6c424827c2ad39a8d09cbdbb8894f915c/Bilder/ArgoCD.png)

### Helm
#### Theorie
Helm-Charts vereinfachen die Bereitstellung und Verwaltung von Anwendungen in OpenShift, da OpenShift auf Kubernetes basiert. Sie bieten eine standardisierte Methode zur Definition von Anwendungen und deren Ressourcen, was die Automatisierung und Wartung von Deployments erleichtert. Helm-Charts ermöglichen eine effiziente und skalierbare Anwendungsbereitstellung in OpenShift-Umgebungen und unterstützen die Optimierung von Entwicklungs- und Betriebsprozessen.

#### Praxis
Da wir Helm bereits installiert haben, können wir direkt mit der Erstellung und Anpassung der Ordnerstruktur für unser Helm-Chart beginnen. Hier ist die grundlegende Ordnerstruktur für ein Helm-Chart:

![image](https://github.com/SilvioM04/M300/blob/8ce9370a4db81e89591fd07b34c1d895ed205521/Bilder/Helm.png)

1. **Wir ersetzen "unsere-webapp" mit dem gewünschten Namen für unser Helm-Chart.**
2. **Chart.yaml**: Enthält Metadaten zu unserem Helm-Chart, wie den Namen, die Version und eine Beschreibung der Anwendung.
3. **values.yaml**: Definiert die Standardwerte für die Ressourcen unserer Anwendung, wie z.B. Image-Name, -Tag und Umgebungsvariablen.
4. **templates/**: Enthält die Ressourcen-Templates für unsere Anwendung. Wir sollten die folgenden YAML-Dateien erstellen bzw. anpassen:
   a. **deployment.yaml**: Definiert das Kubernetes-Deployment für unsere Anwendung, z.B. die Container-Image, Ports und Umgebungsvariablen.
   b. **service.yaml**: Definiert den Kubernetes-Service für unsere Anwendung, der den Zugriff auf unsere Anwendung innerhalb des Clusters ermöglicht.
   c. **route.yaml**: Definiert die OpenShift-Route für unsere Anwendung, die den externen Zugriff auf unsere Anwendung ermöglicht.
   d. **_helpers.tpl**: Enthält wiederverwendbare Template-Funktionen, die in den anderen Ressourcen-Templates verwendet werden können.

In diesen Dateien verwenden wir die Helm-Template-Sprache, um Werte aus der values.yaml-Datei in unseren Ressourcen-Templates einzubinden. Dadurch wird die Anpassung und Aktualisierung unserer Anwendung einfacher.

Nachdem wir die Ordnerstruktur erstellt und angepasst haben, können wir das Helm-Chart in unser BitBucket-Repository integrieren und ArgoCD verwenden, um unsere Anwendung auf OpenShift bereitzustellen.

![image](https://github.com/SilvioM04/M300/blob/8ce9370a4db81e89591fd07b34c1d895ed205521/Bilder/Helm1.png)
![image](https://github.com/SilvioM04/M300/blob/8ce9370a4db81e89591fd07b34c1d895ed205521/Bilder/Helm2.png)
![image](https://github.com/SilvioM04/M300/blob/8ce9370a4db81e89591fd07b34c1d895ed205521/Bilder/Helm3.png)

### ConfigMaps
#### Theorie
In einer Website mit mehreren HTML-Dateien, einer CSS-Datei und Python-Skripten ermöglicht eine OpenShift ConfigMap das Speichern und Verwalten von konfigurierbaren Einstellungen wie Datenbankparametern oder API-Schlüsseln. Dadurch wird die Trennung von Code und Konfiguration verbessert, was die Flexibilität und Wartung der Anwendung erleichtert.

#### Praxis
![image](https://github.com/SilvioM04/M300/blob/8ce9370a4db81e89591fd07b34c1d895ed205521/Bilder/index-html.png)

Hier mussten wir nur den Namen, Namespace (so heisst das Projekt welches wir erstellt hatten), und unter «data:» dann angeben das es das index.html ist. Unter dem index.html mussten wir nur noch eingeben wie die Seite aussehen soll wie in einem Normalen HTML.

Die restlichen ConfigMaps sind im Ordner Bilder https://github.com/SilvioM04/M300/blob/8ce9370a4db81e89591fd07b34c1d895ed205521/Bilder abgelegt, da diese gleich aufgebaut sind.

## Testing-Files
Leider können wir keine Testfälle aufschreibe da sie sensible Daten beinhalten.

![image](https://github.com/SilvioM04/M300/blob/8ce9370a4db81e89591fd07b34c1d895ed205521/Bilder/Website1.png)

![image](https://github.com/SilvioM04/M300/blob/8ce9370a4db81e89591fd07b34c1d895ed205521/Bilder/Website2.png)

![image](https://github.com/SilvioM04/M300/blob/8ce9370a4db81e89591fd07b34c1d895ed205521/Bilder/Website3.png)

# Reflexion

## Persönlicher-Wissenstand

**Helm** ist ein Paketmanager für Kubernetes, der dabei hilft, Anwendungen und Services in Kubernetes-Clustern zu definieren, zu installieren und zu aktualisieren. Helm-Charts sind strukturierte Pakete, die die erforderlichen Komponenten und Ressourcen enthalten, um eine Anwendung in einem Kubernetes-Cluster bereitzustellen. Sie vereinfachen die Bereitstellung und Verwaltung von Anwendungen und können leicht versioniert und geteilt werden.

**ArgoCD** ist ein kontinuierliches Bereitstellungstool für Kubernetes, das sich auf GitOps-Prinzipien stützt. Es ermöglicht, dass die gewünschte Anwendungsstruktur direkt aus einem Git-Repository synchronisiert wird. ArgoCD unterstützt automatische Synchronisation, Rollbacks und eine Vielzahl von Anwendungs- und Umgebungskonfigurationen, um sicherzustellen, dass der Clusterzustand immer dem im Repository definierten Zustand entspricht.

**OpenShift** ist eine Kubernetes-Distribution von Red Hat, die zusätzliche Funktionen und Sicherheit bietet. Es bietet eine umfassende Plattform für die Entwicklung, Bereitstellung und Verwaltung von Container-basierten Anwendungen in der Cloud oder On-Premises. OpenShift erweitert die Möglichkeiten von Kubernetes durch die Integration von zusätzlichen Tools, wie Source-to-Image (S2I) und Operators, um den Entwicklungsprozess zu erleichtern und zu automatisieren.

**BitBucket** ist ein webbasiertes Versionskontrollsystem, das auf Git und Mercurial basiert und von Atlassian entwickelt wurde. Es ermöglicht die Zusammenarbeit an Code, die Verwaltung von Repositories, Code-Reviews und die Integration mit Continuous Integration und Continuous Deployment (CI/CD) Tools. BitBucket bietet sowohl Cloud- als auch Server-basierte Lösungen und lässt sich gut in die Atlassian-Produktfamilie (Jira, Confluence usw.) integrieren.

**ConfigMaps** sind ein Kubernetes-Objekt, das genutzt wird, um konfigurationsbezogene Daten in einer Key-Value-Struktur zu speichern. In OpenShift werden ConfigMaps verwendet, um Anwendungen von Umgebungsvariablen oder Konfigurationsdateien zu trennen. Sie ermöglichen die zentrale Verwaltung von Konfigurationen und können zur Laufzeit in Container gemountet oder als Umgebungsvariablen injiziert werden, wodurch die Notwendigkeit von Neuerstellung von Container-Images bei Konfigurationsänderungen reduziert wird.


# Wissenszuwachs
Was habe ich alles gelernt:
* Openshift kennengelernt
* ArgoCD, BitBucket, Helm Charts und skripting kennengelernt
* Systemumgebung besseres verständnis
* Cloud mit Container verbindung 

# Quellenangaben

| Objekt   |      Quelle     |
|:----------|:-------------|
| Docker-Compose generell | https://docs.docker.com/compose/gettingstarted/ |
| Gute Erklärung zu restart: always | https://serverfault.com/questions/884759/how-does-restart-always-policy-work-in-docker-compose |

