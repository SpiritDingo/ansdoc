# Dockerfile для PHP-сервиса с поддержкой SQL Server, PostgreSQL, Apache2 и MySQL

```dockerfile
# Используем официальный образ PHP с Apache
FROM php:8.2-apache

# Установка системных зависимостей
RUN apt-get update && apt-get install -y \
    gnupg \
    unixodbc \
    unixodbc-dev \
    libpq-dev \
    libzip-dev \
    libicu-dev \
    libonig-dev \
    && rm -rf /var/lib/apt/lists/*

# Установка Microsoft ODBC Driver for SQL Server
RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
    && curl https://packages.microsoft.com/config/debian/$(lsb_release -rs)/prod.list > /etc/apt/sources.list.d/mssql-release.list \
    && apt-get update \
    && ACCEPT_EULA=Y apt-get install -y msodbcsql18 \
    && apt-get install -y unixodbc-dev

# Установка расширений PHP
RUN docker-php-ext-install \
    pdo \
    pdo_mysql \
    mysqli \
    pgsql \
    pdo_pgsql \
    intl \
    mbstring \
    zip \
    && pecl install sqlsrv pdo_sqlsrv \
    && docker-php-ext-enable sqlsrv pdo_sqlsrv

# Настройка Apache
RUN a2enmod rewrite
COPY ./000-default.conf /etc/apache2/sites-available/000-default.conf

# Копирование файлов приложения
COPY . /var/www/html/

# Установка прав
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html

# Порт для Apache
EXPOSE 80

# Запуск Apache в foreground
CMD ["apache2-foreground"]
```

## Конфигурационный файл Apache (000-default.conf)

Создайте файл `000-default.conf` в той же директории, что и Dockerfile:

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

## Сборка и запуск контейнера

1. Соберите образ:
```bash
docker build -t php-multi-db-service .
```

2. Запустите контейнер:
```bash
docker run -d -p 8080:80 --name php-app php-multi-db-service
```

## Дополнительные настройки

1. Для PHP 5.x (если действительно требуется):
   ```dockerfile
   FROM php:5.6-apache
   # Остальные команды могут потребовать адаптации для старой версии PHP
   ```

2. Для добавления MySQL клиента:
   ```dockerfile
   RUN apt-get install -y mysql-client
   ```

3. Для настройки php.ini:
   ```dockerfile
   COPY php.ini /usr/local/etc/php/conf.d/custom.ini
   ```

Этот Dockerfile предоставляет среду с:
- PHP 8.2 (можно изменить на нужную версию)
- Поддержкой SQL Server через sqlsrv и pdo_sqlsrv
- Поддержкой PostgreSQL через pgsql и pdo_pgsql
- Поддержкой MySQL через mysqli, pdo_mysql
- Apache2 с включенным mod_rewrite
- Дополнительными полезными расширениями (intl, mbstring, zip)