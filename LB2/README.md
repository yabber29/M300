# LB2 Dokumentation
![image](https://github.com/yabber29/M300/blob/63db2126507e79fa084f5787fc82990f69c19111/Bilder/Vagrant.png)
============================================================================================================

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
$ vagrant init generic/ubuntu1804 # Vagrantfile wird mit der Box Ubuntu 18.04 erstellt
$ vagrant up # Die Virtuelle Maschine wird gestartet
```
Damit ich nicht die ganze Zeit das Vagrantfile anpassen muss kann man auch mit:
```
$ vagrant ssh # SSH verbindung 
```
auf die VM zugreifen

Doch auch diese Befehle müsst ihr kennen:
```
$ vagrant halt # Schaltet die VM aus
$ vagrant status # Zeigt den aktuellen Status der VM an
$ vagrant destroy # Löscht die VM
```

# Vagrant Code

Um besser zu verstehen, welcher Teil was macht, habe ich denn Code analysiert und auskommentiert:

```
time_zone = "Europe/Zurich" # Zeitzone gesetzt

Vagrant.configure("2") do |config| # Konfiguration der Vagrant-Box
  config.vm.box = "generic/ubuntu1804" # Verwendet die Ubuntu 18.04-Box
  config.vm.network "forwarded_port", guest: 80, host: 2123, auto_correct: true # Weiterleitungsport von 80 auf 2123
  config.ssh.forward_agent = true
  config.vm.synced_folder ".", "/vagrant", type: "virtualbox", # Synchronisiere das aktuelle Verzeichnis mit dem /vagrant-Verzeichnis in der VM
  owner: "www-data", group: "www-data" # Gruppen und Benutzer die Folder festlegen

  config.vm.provider "virtualbox" do |vb| # Provider Konfiguration
    vb.name = "m300_lb2" # VM Name
    vb.memory = 2048 # RAM in MB
    vb.customize ["guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 10000]
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
  end


config.vm.provision "shell" do |s|
    s.args = [time_zone]
    s.inline = <<-SHELL
    
# Konfiguration und installation von Software 

TIME_ZONE=$1
export DEBIAN_FRONTEND=noninteractive # Abfragen für manuelle Benutzerinteraktionen deaktivieren
timedatectl set-timezone "$TIME_ZONE" # Aktualisierung der Zeitzoney<2
apt-get update -q # Updated ganzes System
apt-get isntall -y ufw # Firewall installation
sudo ufw --force enable
sudo ufw allow from 10.0.2.2 to any port 22 # Port 22 für 10.0.2.2 freigeschaltet
sudo ufw allow from 10.0.2.2 to any port 80 # Port 80 für 10.0.2.2 freischlaten
sudo systemctl start ssh
apt-get install -q -y vim git # Vim und Git installieren ins /var/www
apt-get install -q -y apache2 # Installieren weiterer Software
apt-get install -q -y php7.2 libapache2-mod-php7.2
apt-get install -q -y php7.2-curl php7.2-gd php7.2-mbstring php7.2-mysql php7.2-xml php7.2-zip php7.2-bz2 php7.2-intl
apt-get install -q -y mariadb-server mariadb-client
a2enmod rewrite headers
systemctl restart apache2 # Startet den Dienst neu

# Vagrant-Ordner als Apache-Root-Ordner festlegen und dorthin wechseln
dir='/vagrant/www'
if [ ! -d "$dir" ]; then
  mkdir "$dir"
fi
if [ ! -L /var/www/html ]; then
  rm -rf /var/www/html
  ln -fs "$dir" /var/www/html
fi
cd "$dir"
# Vhost konfigurieren
file='/etc/apache2/sites-available/dev.conf'
if [ ! -f "$file" ]; then
  SITE_CONF=$(cat <<EOF
<Directory /var/www/html>
  AllowOverride All
  Options +Indexes -MultiViews +FollowSymLinks
  AddDefaultCharset utf-8
  SetEnv ENVIRONMENT "development"
  php_flag display_errors On
  EnableSendfile Off
</Directory>

EOF
)

file='/etc/apache2/sites-available/001-reverseproxy.conf'
if [ ! -f "$file" ]; then
  Proxy_CONF=$(cat <<EOF
  listen 80
<VirtualHost *:80>
  ProxyPass "/proxy" "http://localhost:2123/"
  ProxyPassReverse "/proxy" "http://localhost:2123/"
</VirtualHost>
EOF

  )

  echo "$SITE_CONF" > "$file"
  echo "$Proxy_CONF" > "$file"
fi
a2ensite dev
a2ensite 001-reverseproxy
a2enmod Proxy
a2enmod Proxy_http
systemctl reload apache2

# Composer installieren
EXPECTED_SIGNATURE="$(wget -q -O - https://composer.github.io/installer.sig)"
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
ACTUAL_SIGNATURE="$(php -r "echo hash_file('SHA384', 'composer-setup.php');")"
if [ "$EXPECTED_SIGNATURE" != "$ACTUAL_SIGNATURE" ]; then
  >&2 echo 'ERROR: Invalid installer signature (Composer)'
  rm composer-setup.php
else
  php composer-setup.php --quiet
  rm composer-setup.php
  mv composer.phar /usr/local/bin/composer
  chmod +x /usr/local/bin/composer
  sudo -H -u vagrant bash -c 'composer global require hirak/prestissimo'
fi

# phpinfo script 
file='phpinfo.php'
if [ ! -f "$file" ]; then
  echo '<?php phpinfo();' > "$file"
fi
# OPcache gui script herunterladen von Webseite
file='opcache.php'
if [ ! -f "$file" ]; then
  wget -nv -O "$file" https://raw.githubusercontent.com/amnuts/opcache-gui/master/index.php
fi
# Adminer script herunterladen von Webseite
file='adminer.php'
if [ ! -f "$file" ]; then
  wget -nv -O "$file" http://www.adminer.org/latest.php
  wget -nv https://raw.githubusercontent.com/vrana/adminer/master/designs/pepa-linha/adminer.css
fi
# Installiren von node, npm, bower, yarn und Webseiteninhalt anzeigen
apt-get install curl
curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
apt-get install -y nodejs
npm install -g bower
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt-get update && sudo apt-get install yarn

#Login für Adminer fürs GUI
sudo mysql <<-EOF
  create user 'ramon'@'localhost' identified by 'Admin1234!!';
  Grant all privileges on *.* to 'ramon'@'localhost';
  Flush privileges;
EOF

SHELL
  end
end
```

**Skirpt Ende**

# Testing

Hier sieht man die Index Seite auf dem Apache Server von http://localhost:2123

![image](https://github.com/yabber29/M300/blob/66f0f9d3b7f44179aef41a074812d492bebe1d05/Bilder/index.png)

## Adminer
Nachdem man sich erfolgreich auf dem GUI mit den gegebenen Credentials angemeldet hat, ist man in der Lage seine Datenbank durch Adminer zu verwalten. Sehr praktisch und nützlich für DB-Admins.

![image](https://github.com/yabber29/M300/blob/4d5b176d63775be84893ac78a538035ec0d48d61/Bilder/Adminer.png)

## OPcache
OPcache ist primär für die Systemüberwachung konzipiert worden. Zu sehen sind nützliche und wichtige Informationen zusammengetragen auf einer Seite.

![image](https://github.com/yabber29/M300/blob/4d5b176d63775be84893ac78a538035ec0d48d61/Bilder/opcache.png)

## Firewall Rules
Die Firewall Rules wurden so gesetzt das nur der Host auf die VM zugreifen kann also IP auf any zu Port 22 (SSH), und Port 80 (http) wurde für alle aufgemacht.

![image](https://github.com/yabber29/M300/blob/5891e8600685b5128808836a92efe26c0cd16e7f/Bilder/Firewall-rules.png)

## Reverse Proxy
Der Output zeigt dies an:

![image](https://github.com/yabber29/M300/blob/fe7087e56d20a709f7435e3dd351eb8004e95c80/Bilder/Reverse-Proxy.png)

Dies gibt an, dass die Verbindung erfolgreich war (HTTP/1.1 200 OK) und dass der Server Apache/2.4.29 (Ubuntu) verwendet wird. Die Antwort enthält auch Informationen zur letzten Änderung der angeforderten Ressource (Last-Modified) und eine ETag, die für Caching-Zwecke verwendet wird.

# Persönlicher Wissenstand

**Git** ist ein verteiltes Versionskontrollsystem, das Entwicklern ermöglicht, gemeinsam an Code-Projekten zu arbeiten und Änderungen zu verfolgen. Es ist besonders nützlich bei der Zusammenarbeit an großen Projekten oder bei der Verwaltung von Änderungen an Software-Code.

**Vagrant** ist ein Tool zur Erstellung und Verwaltung von virtuellen Entwicklungsumgebungen. Es erleichtert Entwicklern das Einrichten von Umgebungen mit spezifischen Software- und Betriebssystemanforderungen, indem es automatisch virtuelle Maschinen erstellt und die benötigte Software installiert.

**Cloud Computing** bezieht sich auf die Bereitstellung von IT-Ressourcen und -Diensten über das Internet. Es umfasst Infrastruktur, Plattformen und Software, die von einem Drittanbieter bereitgestellt und verwaltet werden. Cloud Computing ist insbesondere für Unternehmen und Organisationen nützlich, die skalierbare und flexible IT-Infrastruktur benötigen, ohne dafür eine eigene Infrastruktur aufbauen zu müssen.

**IaC** (Infrastructure as Code) ist eine Methode zur Automatisierung der Infrastrukturverwaltung. Es beinhaltet die Verwendung von Code, um Infrastrukturkomponenten wie Server, Netzwerke und Datenbanken zu erstellen und zu verwalten. Dadurch können Entwickler und IT-Teams Infrastruktur schneller und effizienter bereitstellen und verwalten.

**Firewall** und **Reverse Proxy** sind Sicherheitsmechanismen zum Schutz von Netzwerken und Anwendungen. Eine Firewall blockiert unerwünschten Datenverkehr, während ein Reverse Proxy-Server als Vermittler zwischen Clientanwendungen und Servern fungiert. Beide Mechanismen sind wichtig, um Netzwerke und Anwendungen vor Bedrohungen wie Hackerangriffen zu schützen.

**SSH**(Secure Shell) ist ein Protokoll zur sicheren Remote-Verbindung zu einem anderen Computer oder Server. Es ermöglicht Benutzern, Befehle auszuführen und Dateien sicher zu übertragen. SSH ist ein wichtiger Sicherheitsmechanismus bei der Verwaltung von Netzwerken und Servern.

**Benutzerverwaltung** bezieht sich auf die Verwaltung von Benutzerkonten in einem Computersystem oder einer Anwendung. Es umfasst die Erstellung, Änderung und Löschung von Benutzerkonten sowie die Zuweisung von Zugriffsrechten. Die Verwaltung von Benutzerkonten ist wichtig, um sicherzustellen, dass nur autorisierte Benutzer auf das System zugreifen können.

**Authentifizierung** und **Autorisierung** sind Mechanismen zur Sicherstellung der Identität und Berechtigung von Benutzern in einem Computersystem oder einer Anwendung. Die Authentifizierung stellt sicher, dass der Benutzer tatsächlich der ist, für den er sich ausgibt, während die Autorisierung festlegt, welche Aktionen ein Benutzer ausführen kann, sobald er authentifiziert ist. Diese Mechanismen sind besonders wichtig, um den Zugriff auf sensible Daten und Systeme zu kontrollieren und sicherzustellen, dass nur autorisierte Benutzer darauf zugreifen können.

# Reflexion

**Tag 1:**
Der erste Tag war ziemlich theoretisch, doch konnten wir zumindest einige Einblicke in das Modul bekommen. Der Einstieg war klar und ich wusste bei jedem Thema schon ein wenig.
Die ersten Aufträge fand ich gut, da man die Systeme ausprobieren konnte und sich einen Überblick schaffen konnte. Doch einige Probleme gab es schon, zum Beispiel bei der Installation einer VM von Hand kamen einige Fehler auf mich zu, welche ich aber mit ein wenig Recherche im Internet herausfinden und lösen konnte. Das zweite Problem war bei der Installation einer VM per Vagrant, da machte mir der Antivirus einen Strich durch die Rechnung, die Domain musste ich in die trusted Domains reinschreiben, damit es erkannt wird. Der Rest funktionierte perfekt und ich konnte einiges neues dazu lernen.

**Tag2:**
Durch die Repetitionen am Anfang der Stunde konnte ich mich wieder gut an das letzte Mal erinnern. Heute hatte ich mich um die Dokumentation gekümmert, sodass ich wenigstens etwas darin hatten. Für mich war Markdown etwas neues und ich musste alles neu erlernen. Durch einige Tutorial fiel mir die Einführung leichter, doch sie war nicht reibungslos. Bei der Aufgabe vom Host gerät auf die Localhost Webseite zuzugreifen, per portforwarder hatte ich einige Fehler und Probleme, doch alle Probleme konnte ich lösen und am Schluss funktionierte alles. Dies funktionierte, da ich einige Sachen im Internet gefunden und ausprobiert hatte. Mit der Anleitung vom Modul bin ich nicht 100% schlau geworden, doch ich fand einen Weg. Doch aus diesem Tag konnte ich einiges mitnehmen.

**Tag 3:**
Heute fing ich an Vagrant ein wenig mehr zu verstehen wie die Befehle funktionieren aber auch wie das Vagrantfile mit der VM zusammenhängt. Heute wollte ich auch einige neue Boxen ausprobieren um zu sehen, ob mir ein andere OS besser liegt. Bei diesem Punkt hatte ich Packer angeschaut, welches spannend war doch bei mir nicht ganz funktioniert hatte, doch die Funktionsweise wurde mir klar. Heute hatte ich mir auch einige Ideen zu meiner LB2 gemacht doch dies war nicht so einfach wie es schien. Als der Tag zu Ende ging dokumentierte ich das gelernte und lud alles, was ich hatte auf mein GitHub Account hoch.

**Tag 4:**
Der heutige Schwerpunkt lag bei der Sicherheit der VM, darunter gehört eine Firewall / Reverse Proxy, Rechteverwaltung, SSH und Autorisierung wie Authentifizierung. Der heutige Tag gefiel mir ziemlich gut da, ich keine grosse Probleme hatte bei der Durchführung der eben genannten Sicherheitsaspekten. Zudem fing ich heute an meine Idee umzusetzen und ein Vagrantfile zu erstellen. Durch das Internet und Foren konnte ich einige physikalische Probleme überwältigen, Bsp. Wenn ich ''Vagrant up'' machte, hielt es immer bei SSH auth method: private key, durch das Forum wusste ich, dass ich in meinem Virenschutz eine weitere Domain eintragen musste, danach lief alles wie zuvor. Da ich heute mehr für die Sicherheit aufgewandt habe, arbeitete ich Zuhause noch weiter.

# Wissenszuwachs

# Quellen

Beschreibung | Quelle
Vagrant.png | https://www.google.com/url?sa=i&url=https%3A%2F%2Fgithub.com%2Ftopics%2Fvagrant&psig=AOvVaw0SOtFeHJVQ3lKXCpqD9XtK&ust=1679145287107000&source=images&cd=vfe&ved=0CBAQjRxqFwoTCPCP3P-F4_0CFQAAAAAdAAAAABAj

Restliche Bilder | Selbst gemacht