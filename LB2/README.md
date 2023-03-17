# LB2 Dokumentation
![image](https://github.com/yabber29/M300/blob/63db2126507e79fa084f5787fc82990f69c19111/Bilder/Vagrant.png)
============================================================================================================
<<<<<<< HEAD

# Inhaltsverzeichnis
1. [Einführung](#Einführung)
2. [Vagrantfile](#Vagrant)
3. [Testing](#Testing)
4. [Quellen](#Quellen)

# Einführung

Da viele einen Webserver mit einer Datenbank machten, wollten wir etwas leicht "spezielleres" machen. Im Frontend arbeitet ein Apache2 Webserver, der die Dienste **Adminer und OPcache** in einem GUI ersichtlich machen sollte. Im Hintergrund laufen PHP und MariaDB, damit die Services ordnungsgemäss funktionieren. All dies sollte via Portweiterleitung ereichbar sein.

Damit ich mir das ganze besser vorstellen konnte erstellte ich zuerst ein Netzwerkplan.
![image](https://github.com/yabber29/M300/blob/54e7578466e0a3a050328bc73c797b3ee4fdae15/Bilder/Netzwerkplan.png)

Auf dem Hostsystem können wir http://localhost:2123 eingeben. Anschliessend leitet dieser die Anfrage an die VM auf Port 80 weiter. Wir sollten in der Lage sein, die Services so nutzen zu können.

Damit ich die VM per Vagrant erstellen konnte brauchte ich zwingend diese zwei Befehle:
```
$ vagrant init generic/ubuntu1804 #Vagrantfile wird mit der Box Ubuntu 18.04 erstellt
$ vagrant up #Die Virtuelle Maschine wird gestartet
```
Damit ich nicht die ganze Zeit das Vagrantfile anpassen muss kann man auch mit:
```
$ vagrant ssh #SSH verbindung 
```
auf die VM zugreifen

# Vagrant Code

Um besser zu verstehen, welcher Teil was macht, habe ich denn Code analysiert und auskommentiert:


