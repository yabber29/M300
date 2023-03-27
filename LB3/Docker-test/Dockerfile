# Verwenden Sie das offizielle Nextcloud-Image als Basis
FROM nextcloud:latest

# Installieren Sie zusätzliche Pakete
RUN apt-get update && apt-get install -y \
    vim \
    && rm -rf /var/lib/apt/lists/*

# Kopieren Sie eine lokale Konfigurationsdatei in den Container
COPY config.php /var/www/html/config/

# Öffnen Sie die Ports 80 und 443 für eingehenden Verkehr
EXPOSE 80 443

# Starten Sie Nextcloud, wenn der Container gestartet wird
CMD ["apache2-foreground"]
