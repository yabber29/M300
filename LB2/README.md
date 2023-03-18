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



# Quellen

Beschreibung | Quelle
Vagrant.png | https://www.google.com/url?sa=i&url=https%3A%2F%2Fgithub.com%2Ftopics%2Fvagrant&psig=AOvVaw0SOtFeHJVQ3lKXCpqD9XtK&ust=1679145287107000&source=images&cd=vfe&ved=0CBAQjRxqFwoTCPCP3P-F4_0CFQAAAAAdAAAAABAj

Restliche Bilder | Selbst gemacht