# Docker Compose для PHP-сервиса с SQL Server, PostgreSQL, Apache2 и MySQL

Вот комплексное решение с Dockerfile и docker-compose.yml для развертывания PHP-сервиса с поддержкой всех указанных СУБД.

## 1. Dockerfile

```dockerfile
# Используем официальный образ PHP 8.2 с Apache (для PHP 5.x замените на php:5.6-apache)
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
    libxml2-dev \
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
    soap \
    && pecl install sqlsrv pdo_sqlsrv \
    && docker-php-ext-enable sqlsrv pdo_sqlsrv

# Настройка Apache
RUN a2enmod rewrite
COPY ./apache-config.conf /etc/apache2/sites-available/000-default.conf

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

## 2. Apache Config (apache-config.conf)

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/public

    <Directory /var/www/html/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
        FallbackResource /index.php
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

## 3. docker-compose.yml

```yaml
version: '3.8'

services:
  php-app:
    build: .
    ports:
      - "8080:80"
    volumes:
      - ./:/var/www/html
    depends_on:
      - db-mysql
      - db-postgres
      - db-sqlserver
    environment:
      - DB_MYSQL_HOST=db-mysql
      - DB_MYSQL_DATABASE=app_db
      - DB_MYSQL_USER=root
      - DB_MYSQL_PASSWORD=secret
      - DB_PGSQL_HOST=db-postgres
      - DB_PGSQL_DATABASE=app_db
      - DB_PGSQL_USER=postgres
      - DB_PGSQL_PASSWORD=postgres
      - DB_SQLSRV_HOST=db-sqlserver
      - DB_SQLSRV_DATABASE=app_db
      - DB_SQLSRV_USER=sa
      - DB_SQLSRV_PASSWORD=YourStrong!Passw0rd

  db-mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: app_db
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql

  db-postgres:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_db
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

  db-sqlserver:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      SA_PASSWORD: YourStrong!Passw0rd
      ACCEPT_EULA: Y
    ports:
      - "1433:1433"
    volumes:
      - sqlserver-data:/var/opt/mssql

volumes:
  mysql-data:
  postgres-data:
  sqlserver-data:
```

## 4. Инструкции по использованию

1. Создайте структуру проекта:
```
project/
├── docker-compose.yml
├── Dockerfile
├── apache-config.conf
└── src/
    └── public/
        └── index.php
```

2. Соберите и запустите сервисы:
```bash
docker-compose up -d --build
```

3. Проверьте работу:
- PHP приложение: http://localhost:8080
- MySQL: порт 3306
- PostgreSQL: порт 5432
- SQL Server: порт 1433

## 5. Пример файла index.php для проверки подключений

```php
<?php
// Проверка MySQL подключения
try {
    $mysql = new PDO(
        "mysql:host=" . getenv('DB_MYSQL_HOST') . ";dbname=" . getenv('DB_MYSQL_DATABASE'),
        getenv('DB_MYSQL_USER'),
        getenv('DB_MYSQL_PASSWORD')
    );
    $mysqlStatus = "MySQL: Connection OK";
} catch (PDOException $e) {
    $mysqlStatus = "MySQL: " . $e->getMessage();
}

// Проверка PostgreSQL подключения
try {
    $pgsql = new PDO(
        "pgsql:host=" . getenv('DB_PGSQL_HOST') . ";dbname=" . getenv('DB_PGSQL_DATABASE'),
        getenv('DB_PGSQL_USER'),
        getenv('DB_PGSQL_PASSWORD')
    );
    $pgsqlStatus = "PostgreSQL: Connection OK";
} catch (PDOException $e) {
    $pgsqlStatus = "PostgreSQL: " . $e->getMessage();
}

// Проверка SQL Server подключения
try {
    $sqlsrv = new PDO(
        "sqlsrv:Server=" . getenv('DB_SQLSRV_HOST') . ";Database=" . getenv('DB_SQLSRV_DATABASE'),
        getenv('DB_SQLSRV_USER'),
        getenv('DB_SQLSRV_PASSWORD')
    );
    $sqlsrvStatus = "SQL Server: Connection OK";
} catch (PDOException $e) {
    $sqlsrvStatus = "SQL Server: " . $e->getMessage();
}
?>
<!DOCTYPE html>
<html>
<head>
    <title>PHP Multi-DB Test</title>
</head>
<body>
    <h1>Database Connection Status</h1>
    <ul>
        <li><?= $mysqlStatus ?></li>
        <li><?= $pgsqlStatus ?></li>
        <li><?= $sqlsrvStatus ?></li>
    </ul>
</body>
</html>
```

Этот комплект предоставляет полноценную среду разработки с:
- PHP 8.2 + Apache2
- Поддержкой MySQL (mysqlnd)
- Поддержкой PostgreSQL (php5-pgsql)
- Поддержкой SQL Server (sqlsrv)
- Автоматической настройкой всех СУБД через Docker Compose
- Персистентным хранением данных (volumes)