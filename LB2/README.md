# LB2 Dokumentation
![image](https://github.com/yabber29/M300/blob/63db2126507e79fa084f5787fc82990f69c19111/Bilder/Vagrant.png)
============================================================================================================

# Inhaltsverzeichnis
1. [Einführung](#Einführung)
2. [Vagrantfile](#Vagrant)
3. [Testing](#Testing)
4. [Persönlicher Wissenstand](#Persönlicher-Wissenstand)
5. [Reflexion](#Reflexion)
6. [Wissenszuwachs](#Wissenszuwachs)
7. [Quellen](#Quellen)

# Einführung

Da viele einen Webserver mit einer Datenbank machten, will ich etwas leicht "spezielleres" machen. Im Frontend arbeitet ein Apache2 Webserver, der die Dienste **Adminer und OPcache** in einem GUI ersichtlich machen sollte. Im Hintergrund laufen PHP und MariaDB, damit die Services ordnungsgemäss funktionieren. All dies sollte via Portweiterleitung ereichbar sein.

Damit ich mir das ganze besser vorstellen konnte erstellte ich zuerst ein Netzwerkplan.
![image](https://github.com/yabber29/M300/blob/54e7578466e0a3a050328bc73c797b3ee4fdae15/Bilder/Netzwerkplan.png)

Auf dem Hostsystem kann ich http://localhost:2123 eingeben. Anschliessend leitet dieser die Anfrage an die VM auf Port 80 weiter. Ich sollte in der Lage sein, die Services so nutzen zu können.

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
time_zone = "Europe/Zurich"

Vagrant.configure("2") do |config| # Konfiguration der Vagrant-Box
  config.vm.box = "generic/ubuntu1804" # Verwendet die Ubuntu 18.04-Box
  config.vm.network "forwarded_port", guest: 80, host: 2123, auto_correct: true # Weiterleitungsport von 80 auf 2123
  config.ssh.forward_agent = true
  config.vm.synced_folder ".", "/vagrant", type: "virtualbox", # Synchronisiere das aktuelle Verzeichnis mit dem /vagrant-Verzeichnis in der VM
  owner: "www-data", group: "www-data" # Gruppen und Benutzer die Folder festlegen

  config.vm.provider "virtualbox" do |vb| # Provider Konfiguration
    vb.name = "m300_lb2" # VM Name
    vb.memory = 2048 # RAN in MB
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
timedatectl set-timezone "$TIME_ZONE" # Aktualisierung der Zeitzone
apt-get update -q # Updated ganzes System
apt-get isntall -y ufw # Firewall installation
sudo ufw --force enable
sudo ufw allow from 10.0.2.2 to any port 22 # Port 22 für 10.0.2.2 freigeschaltet
sudo ufw allow 80 # Port 80 für alle freischlaten
sudo systemctl start ssh
apt-get install -q -y vim git # Vim und Git installieren ins /var/www
apt-get install -q -y apache2 # Installieren weiterer Software
apt-get install -q -y php7.2 libapache2-mod-php7.2
apt-get install -q -y php7.2-curl php7.2-gd php7.2-mbstring php7.2-mysql php7.2-xml php7.2-zip php7.2-bz2 php7.2-intl
apt-get install -q -y mariadb-server mariadb-client
apt-get install -y ldap-utils
a2enmod rewrite headers
ldapsearch -x -LLL -H ldap://your-ldap-server -D "cn=admin,dc=example,dc=com" -w yourpassword -b "ou=users,dc=example,dc=com" "(objectclass=*)" cn email
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
sudo a2enmod lbmethod_byrequests
systemctl restart apache2
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
  echo "$SITE_CONF" > "$file"
fi
a2ensite dev
sudo touch /etc/apache2/sites-available/reverse-proxy.conf
sudo bash -c 'cat > /etc/apache2/sites-available/reverse-proxy.conf << EOL
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://127.0.0.1:2123/
ProxyPassReverse / http://127.0.0.1:2123/
ServerName reverse-proxy.example.com
</VirtualHost>
EOL'

# Vhost aktivieren
sudo a2ensite reverse-proxy.conf

# Apache neustarten
sudo service apache2 restart
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

## HTTPS
Zusätzlich wollte ich https aktivieren doch dies gelang mir in der kurzen nicht mehr. Mir wird nur die normale Standart Seite von Apache angezeigt mit HTTPS doch mit meiner Seite funktioniert dies leider nicht.

![image](https://github.com/yabber29/M300/blob/bf85bbf1938eeffb74d2ed972b2183a878be0cab/Bilder/HTTPS.png)


# Persönlicher-Wissenstand

 Die Zusammenarbeit am Code und das Nachverfolgen von Änderungen wird mit **Git** ermöglicht – einem verteilten Versionskontrollsystem.  Bei der Arbeit an großen Projekten oder der Verwaltung von Softwarecode-Updates ist dieses Tool besonders nützlich.

 Die Verwaltung virtueller Entwicklungsumgebungen wird mit **Vagrant** vereinfacht, einem Tool, das die Erstellung virtueller Maschinen und die Installation der erforderlichen Software automatisiert.  Es wurde entwickelt, um Entwicklern das Einrichten von Umgebungen mit spezifischen Anforderungen, wie z. B. Software- und Betriebssystemanforderungen, zu erleichtern.

 IT-Ressourcen und -Dienste, die über das Internet bereitgestellt werden, sind das, worum es bei **Cloud Computing** geht.  Es wird von einem Drittanbieter verwaltet und umfasst Infrastruktur, Software und Plattformen.  Mit Cloud Computing müssen Unternehmen keine eigene Infrastruktur aufbauen, was es ideal für diejenigen macht, die eine flexible und skalierbare IT-Infrastruktur suchen.

 Die Verwendung von Code zum Erstellen und Verwalten von Infrastrukturkomponenten wie Servern, Netzwerken und Datenbanken wird als **IaC** (Infrastructure as Code) bezeichnet.  IaC ist ein innovativer Ansatz zur Automatisierung des Infrastrukturmanagements, der eine schnellere und effizientere Infrastrukturbereitstellung und -verwaltung durch Entwickler und IT-Teams ermöglicht.

 Anwendungen und Netzwerke können durch die Implementierung eines **Reverse-Proxys** oder einer **Firewall** gesichert werden.  Eine Firewall dient dazu, unerwünschten Datenverkehr zu blockieren, und ein Reverse-Proxy-Server ist ein Vermittler zwischen Client-Programmen und Servern.  Es ist wichtig, dass beide Mechanismen vorhanden sind, um sich gegen feindliche Aktivitäten wie Cyberangriffe zu verteidigen.

 **Secure Shell (SSH)** ist ein Protokoll, das für die sichere Fernverbindung mit einem Computer oder Server verwendet wird.  Dieser Mechanismus ermöglicht sichere Dateiübertragungen und erlaubt Benutzern, Befehle auszuführen.  Für die Netzwerk- und Serververwaltung ist SSH ein wichtiges Sicherheitstool.

 In einem Computersystem oder einer Anwendung wird die Verwaltung von Benutzerkonten als **Benutzerverwaltung** bezeichnet, die das Erstellen, Ändern und Entfernen von Benutzerkonten umfasst, während Zugriffsrechte gewährt werden.  Es ist von größter Bedeutung, Benutzerkonten so zu kontrollieren, dass nur authentifizierte Benutzer auf das System zugreifen können.

 Um den Zugriff auf vertrauliche Daten und Systeme zu kontrollieren, ist es wichtig, über Mechanismen zu verfügen, die die Identität und die Berechtigungsstufen von Benutzern auf einem Computersystem oder einer Anwendung bestätigen.  Die beiden primären Mechanismen dafür sind **Authentifizierung** und **Autorisierung**.  Die Authentifizierung überprüft die behauptete Identität des Benutzers und stellt sicher, dass er derjenige ist, für den er sich ausgibt, während die Autorisierung festlegt, welche Aufgaben er nach der Bestätigung ausführen kann.  Nur autorisierte Benutzer sollten Zugriff auf sensible Daten und Systeme haben, die diese Mechanismen wirksam schützen helfen.



# Reflexion

**Tag 1:**
Der erste Tag war ziemlich theoretisch, doch ich konnte zumindest einige Einblicke in das Modul bekommen. Der Einstieg war klar und ich wusste bei jedem Thema schon ein wenig.
Die ersten Aufträge fand ich gut, da man die Systeme ausprobieren konnte und sich einen Überblick schaffen konnte. Doch einige Probleme gab es schon, zum Beispiel bei der Installation einer VM von Hand kamen einige Fehler auf mich zu, welche ich aber mit ein wenig Recherche im Internet herausfinden und lösen konnte. Das zweite Problem war bei der Installation einer VM per Vagrant, da machte mir der Antivirus einen Strich durch die Rechnung, die Domain musste ich in die trusted Domains reinschreiben, damit es erkannt wird. Der Rest funktionierte perfekt und ich konnte einiges neues dazu lernen.

**Tag2:**
Durch die Repetitionen am Anfang der Stunde konnte ich mich wieder gut an das letzte Mal erinnern. Heute hatte ich mich um die Dokumentation gekümmert, sodass ich wenigstens etwas darin hatten. Für mich war Markdown etwas neues und ich musste alles neu erlernen. Durch einige Tutorial fiel mir die Einführung leichter, doch sie war nicht reibungslos. Bei der Aufgabe vom Host gerät auf die Localhost Webseite zuzugreifen, per portforwarder hatte ich einige Fehler und Probleme, doch alle Probleme konnte ich lösen und am Schluss funktionierte alles. Dies funktionierte, da ich einige Sachen im Internet gefunden und ausprobiert hatte. Mit der Anleitung vom Modul bin ich nicht 100% schlau geworden, doch ich fand einen Weg. Doch aus diesem Tag konnte ich einiges mitnehmen.

**Tag 3:**
Heute fing ich an Vagrant ein wenig mehr zu verstehen wie die Befehle funktionieren aber auch wie das Vagrantfile mit der VM zusammenhängt. Heute wollte ich auch einige neue Boxen ausprobieren um zu sehen, ob mir ein andere OS besser liegt. Bei diesem Punkt hatte ich Packer angeschaut, welches spannend war doch bei mir nicht ganz funktioniert hatte, doch die Funktionsweise wurde mir klar. Heute hatte ich mir auch einige Ideen zu meiner LB2 gemacht doch dies war nicht so einfach wie es schien. Als der Tag zu Ende ging dokumentierte ich das gelernte und lud alles, was ich hatte auf mein GitHub Account hoch.

**Tag 4:**
Der heutige Schwerpunkt lag bei der Sicherheit der VM, darunter gehört eine Firewall / Reverse Proxy, Rechteverwaltung, SSH und Autorisierung wie Authentifizierung. Der heutige Tag gefiel mir ziemlich gut da, ich keine grosse Probleme hatte bei der Durchführung der eben genannten Sicherheitsaspekten. Zudem fing ich heute an meine Idee umzusetzen und ein Vagrantfile zu erstellen. Durch das Internet und Foren konnte ich einige physikalische Probleme überwältigen, Bsp. Wenn ich ''Vagrant up'' machte, hielt es immer bei SSH auth method: private key, durch das Forum wusste ich, dass ich in meinem Virenschutz eine weitere Domain eintragen musste, danach lief alles wie zuvor. Da ich heute mehr für die Sicherheit aufgewandt habe, arbeitete ich Zuhause noch weiter.

# Wissenszuwachs
Was habe ich alles gelernt:
* Vagrant kennengelernt
* Firewall und Reverse Proxy auf Ubuntu 
* Automatisierung mit einem File
* Portforwarding von Host aus zugänglich
* Login im vorhinein festlegen


# Quellen

Beschreibung | Quelle
Vagrant.png | https://www.google.com/url?sa=i&url=https%3A%2F%2Fgithub.com%2Ftopics%2Fvagrant&psig=AOvVaw0SOtFeHJVQ3lKXCpqD9XtK&ust=1679145287107000&source=images&cd=vfe&ved=0CBAQjRxqFwoTCPCP3P-F4_0CFQAAAAAdAAAAABAj

Restliche Bilder | Selbst gemacht
