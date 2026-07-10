Ниже — примеры настройки Apache с PHP-FPM (через mod_proxy_fcgi) и основные приёмы работы с HTTP-заголовками: передача входящих заголовков в PHP, проброс авторизации, модификация ответа, проксирование и CORS.

---

1. Базовая связка Apache + PHP-FPM

В современных конфигурациях (Apache 2.4) используют SetHandler или ProxyPassMatch. PHP-скрипты выполняет пул FPM, а Apache выступает фронтендом.

Простой виртуальный хост (файл .conf)

```apache
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/html

    # Передаём все .php файлы в PHP-FPM
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
    </FilesMatch>

    # Или через ProxyPassMatch (альтернативный вариант):
    # ProxyPassMatch ^/(.*\.php)$ unix:/run/php/php8.2-fpm.sock|fcgi://localhost/var/www/html/$1

    <Directory /var/www/html>
        Require all granted
    </Directory>
</VirtualHost>
```

При такой связке Apache автоматически преобразует HTTP-заголовки запроса в переменные окружения FastCGI вида HTTP_*. В PHP они доступны через $_SERVER:

· $_SERVER['HTTP_HOST']
· $_SERVER['HTTP_USER_AGENT']
· $_SERVER['HTTP_X_CUSTOM_HEADER'] и т. д.

---

2. Передача заголовка Authorization (важно!)

По умолчанию Apache не передаёт заголовок Authorization в PHP-FPM по соображениям безопасности. Это ломает любую Basic/Digest/Bearer-авторизацию в PHP.

Решение – включить передачу (Apache 2.4.13+):

```apache
<VirtualHost *:80>
    ServerName api.example.com
    DocumentRoot /var/www/api

    # Передаём заголовок Authorization
    CGIPassAuth On

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
```

Для старых версий Apache можно использовать SetEnvIf:

```apache
SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1
```

Теперь $_SERVER['HTTP_AUTHORIZATION'] в PHP будет заполнен.

---

3. Установка пользовательских заголовков запроса (перед передачей в PHP)

Можно добавить заголовок в Apache до отправки в бэкенд. Например, для передачи реального IP клиента при использовании прокси:

```apache
<VirtualHost *:80>
    ServerName app.example.com

    # Сохраняем IP клиента
    RequestHeader set X-Real-IP "%{REMOTE_ADDR}s"
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"

    # Или добавляем, если уже есть
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s, %{HTTP_X-Forwarded-For}e"

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
```

В PHP они будут доступны как $_SERVER['HTTP_X_REAL_IP'] и $_SERVER['HTTP_X_FORWARDED_FOR'].

---

4. Модификация заголовков ответа (Apache)

Ответ от PHP-FPM можно доработать директивами Header в Apache.

Пример: принудительно убрать заголовок X-Powered-By и добавить CORS

```apache
<VirtualHost *:80>
    ServerName public.api.com
    DocumentRoot /var/www/public

    # Убираем заголовок из ответа PHP
    Header always unset X-Powered-By

    # CORS заголовки (можно задать глобально, не трогая PHP)
    Header always set Access-Control-Allow-Origin "*"
    Header always set Access-Control-Allow-Methods "GET, POST, OPTIONS"
    Header always set Access-Control-Allow-Headers "Authorization, Content-Type"

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php-fpm.sock|fcgi://localhost/"
    </FilesMatch>

    # Обработка preflight запросов OPTIONS
    RewriteEngine On
    RewriteCond %{REQUEST_METHOD} OPTIONS
    RewriteRule ^(.*)$ $1 [R=204,L]
</VirtualHost>
```

Обратите внимание на always – это гарантирует, что заголовок будет добавлен даже для ошибочных ответов (401, 500 и т. д.), а не только при 2xx.

Условная установка заголовков:

```apache
# Добавляем Content-Security-Policy только для HTML
<FilesMatch "\.(html|php)$">
    Header always set Content-Security-Policy "default-src 'self'"
</FilesMatch>

# Кэширование статики, но не PHP
<FilesMatch "\.(css|js|png|jpg|gif|ico)$">
    Header set Cache-Control "max-age=604800, public"
</FilesMatch>
```

---

5. Конфигурация PHP-FPM пула для проброса переменных окружения

Иногда нужно явно задать переменные, которые будут видны в PHP. Это делается в файле пула (например, /etc/php/8.2/fpm/pool.d/www.conf):

```ini
; Пробрасываем кастомные значения
env[APP_ENV] = production
env[MY_CUSTOM_HEADER] = $HTTP_X_CUSTOM_HEADER   ; можно ссылаться на заголовки запроса
```

Но будьте осторожны: переменные, переданные через env, переопределяют значения, пришедшие от Apache. Лучше использовать параметр clear_env = no, чтобы сохранить все переменные окружения, установленные веб-сервером, и дополнить их только необходимыми.

```ini
clear_env = no

; Добавляем дополнительные переменные
env[APPLICATION_MODE] = production
```

---

6. Отладка заголовков

Логирование заголовков в Apache:

```apache
# Логируем выбранные заголовки запроса
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" X-Real-IP:\"%{X-Real-IP}i\"" combined_with_realip
CustomLog /var/log/apache2/access.log combined_with_realip

# Логируем заголовки ответа
Header always note X-Request-ID request_id
LogFormat "... %{request_id}n ..." ...
```

Просмотр заголовков в PHP (отладка):

Временно добавьте скрипт:

```php
<?php
header('Content-Type: text/plain');
print_r($_SERVER);
```

Или используйте getallheaders() (доступна, если PHP работает как Apache module, но для FPM потребуется аналог):

```php
<?php
if (function_exists('getallheaders')) {
    print_r(getallheaders());
} else {
    // эмуляция
    $headers = [];
    foreach ($_SERVER as $name => $value) {
        if (substr($name, 0, 5) == 'HTTP_') {
            $headers[str_replace(' ', '-', ucwords(strtolower(str_replace('_', ' ', substr($name, 5)))))] = $value;
        }
    }
    print_r($headers);
}
```

---

7. Типичные проблемы и их решение

· Заголовок Host не тот – в Apache настройте ProxyPreserveHost On (в контексте ProxyPassMatch), но обычно при SetHandler он передаётся правильно.
· Большие заголовки режутся – увеличьте лимит в пуле FPM: request_terminate_timeout и буферы. Для Apache при использовании mod_proxy_fcgi может потребоваться настройка ProxyFCGIBackendType GENERIC или увеличение LimitRequestFieldSize.
· Заголовки, установленные в PHP (header()), не видны в Apache? Они передаются как есть, Apache может их дополнить через Header append. Если PHP устанавливает X-Foo: bar, а в Apache задано Header always set X-Foo "baz", победит Apache, т. к. он добавляет заголовки после обработки ответа. Используйте append, чтобы добавить новое значение.

---

Итог

Настройка Apache + PHP-FPM для работы с заголовками сводится к нескольким моментам:

1. Базовая передача: включите CGIPassAuth On для авторизации.
2. Модификация запроса: RequestHeader set – передаёт IP, токены и т. д.
3. Корректировка ответа: Header always set/unset/append – CORS, безопасность, кэширование.
4. Переменные окружения: настройка пула FPM (clear_env = no, env[...]).
5. Отладка: формат логов и временный скрипт в PHP.

Приведённые примеры конфигураций работают на Apache 2.4 с модулями proxy, proxy_fcgi, headers, rewrite (включаются при необходимости).