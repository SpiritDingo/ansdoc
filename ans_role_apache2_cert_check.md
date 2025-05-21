# Ansible роль для проверки цепочки сертификатов Apache2

Эта роль предназначена для проверки SSL/TLS сертификатов, используемых в Apache2, включая соответствие между ключом и сертификатом, валидность цепочки и сроки действия.

## Структура роли

```
roles/apache2_cert_check/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
├── templates/
│   └── verify_apache_ssl.sh.j2
└── handlers/
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
# Директория с SSL сертификатами Apache
apache_ssl_dir: "/etc/apache2/ssl"

# Имена файлов сертификатов (от листового к корневому)
apache_certificates:
  - server.crt
  - intermediate.crt
  - ca.crt

# Имя файла с приватным ключом
apache_private_key: "server.key"

# Проверять срок действия
check_expiration: true
expiration_warning_days: 30

# Проверять конфигурацию Apache после проверки сертификатов
verify_apache_config: true
```

### tasks/main.yml

```yaml
---
- name: Установка необходимых пакетов
  apt:
    name:
      - openssl
      - apache2
    state: present

- name: Создание скрипта проверки SSL
  template:
    src: verify_apache_ssl.sh.j2
    dest: /tmp/verify_apache_ssl.sh
    mode: '0755'

- name: Проверка SSL сертификатов Apache
  command: /tmp/verify_apache_ssl.sh
  register: ssl_check
  changed_when: false
  notify:
    - restart apache2

- name: Вывод результатов проверки
  debug:
    var: ssl_check.stdout_lines

- name: Проверка конфигурации Apache
  command: apache2ctl configtest
  register: apache_config_test
  changed_when: false
  when: verify_apache_config

- name: Вывод результатов проверки конфигурации
  debug:
    var: apache_config_test.stdout_lines
```

### templates/verify_apache_ssl.sh.j2

```bash
#!/bin/bash

APACHE_SSL_DIR="{{ apache_ssl_dir }}"
CERTIFICATES=({{ apache_certificates | join(' ') }})
PRIVATE_KEY="{{ apache_private_key }}"
EXPIRATION_DAYS={{ expiration_warning_days }}

# Функции проверки
function check_key_cert_match {
  local key="$APACHE_SSL_DIR/$PRIVATE_KEY"
  local cert="$APACHE_SSL_DIR/${CERTIFICATES[0]}"
  
  if [ ! -f "$key" ]; then
    echo "ERROR: Private key file $key not found"
    return 1
  fi
  
  if [ ! -f "$cert" ]; then
    echo "ERROR: Certificate file $cert not found"
    return 1
  fi
  
  key_modulus=$(openssl rsa -noout -modulus -in "$key" | openssl md5)
  cert_modulus=$(openssl x509 -noout -modulus -in "$cert" | openssl md5)
  
  if [ "$key_modulus" != "$cert_modulus" ]; then
    echo "ERROR: Apache private key $PRIVATE_KEY does not match certificate ${CERTIFICATES[0]}"
    return 1
  else
    echo "OK: Apache private key matches server certificate"
    return 0
  fi
}

function check_cert_expiration {
  local cert="$APACHE_SSL_DIR/$1"
  
  if [ ! -f "$cert" ]; then
    echo "ERROR: Certificate file $cert not found"
    return 1
  fi
  
  end_date=$(openssl x509 -enddate -noout -in "$cert" | cut -d= -f2)
  end_epoch=$(date --date="$end_date" +%s)
  now_epoch=$(date +%s)
  days_left=$(( (end_epoch - now_epoch) / 86400 ))
  
  if [ $days_left -lt 0 ]; then
    echo "ERROR: Certificate $1 expired on $end_date"
    return 1
  elif [ $days_left -lt $EXPIRATION_DAYS ]; then
    echo "WARNING: Certificate $1 expires in $days_left days (on $end_date)"
    return 2
  else
    echo "OK: Certificate $1 valid for $days_left days (until $end_date)"
    return 0
  fi
}

function verify_cert_chain {
  # Создаем временный файл с цепочкой сертификатов
  local chain_file=$(mktemp)
  
  # Объединяем все сертификаты кроме корневого
  for (( i=0; i<${#CERTIFICATES[@]}-1; i++ )); do
    cat "$APACHE_SSL_DIR/${CERTIFICATES[$i]}" >> "$chain_file"
  done
  
  # Проверяем цепочку с корневым сертификатом
  openssl verify -CAfile "$APACHE_SSL_DIR/${CERTIFICATES[-1]}" -untrusted "$chain_file" "$APACHE_SSL_DIR/${CERTIFICATES[0]}"
  local result=$?
  
  rm -f "$chain_file"
  return $result
}

# Основная проверка
ERRORS=0

echo "=== Проверка SSL сертификатов Apache ==="

# Проверка соответствия ключа и сертификата
echo "Проверка соответствия ключа и сертификата..."
check_key_cert_match
ERRORS=$((ERRORS + $?))

# Проверка сроков действия
{% if check_expiration %}
echo "Проверка сроков действия сертификатов..."
for cert in "${CERTIFICATES[@]}"; do
  check_cert_expiration "$cert"
  ERRORS=$((ERRORS + $?))
done
{% endif %}

# Проверка цепочки сертификатов
echo "Проверка цепочки сертификатов..."
verify_cert_chain
ERRORS=$((ERRORS + $?))

exit $ERRORS
```

### handlers/main.yml

```yaml
---
- name: restart apache2
  service:
    name: apache2
    state: restarted
```

## Использование роли

1. Создайте playbook:

```yaml
- hosts: apache_servers
  roles:
    - apache2_cert_check
```

2. Переопределите переменные при необходимости:

```yaml
- hosts: apache_servers
  vars:
    apache_ssl_dir: "/etc/httpd/ssl"
    apache_certificates:
      - httpd.crt
      - chain.crt
    apache_private_key: "httpd.key"
    expiration_warning_days: 15
  roles:
    - apache2_cert_check
```

## Дополнительные возможности

1. Для проверки конкретных виртуальных хостов можно добавить:

```yaml
- name: Проверить SSL конфигурацию виртуальных хостов
  shell: |
    for vhost in $(apache2ctl -S 2>/dev/null | grep "port 443" | awk '{print $NF}'); do
      echo "Checking $vhost"
      openssl s_client -connect localhost:443 -servername $vhost -prexit 2>/dev/null | openssl x509 -noout -dates
    done
  register: vhost_check
  changed_when: false
```

2. Для автоматического обновления просроченных сертификатов можно добавить задачи с использованием ACME клиента (например, certbot).

3. Для мониторинга можно интегрировать вывод в системы мониторинга (Zabbix, Prometheus).

Роль предоставляет комплексную проверку SSL/TLS конфигурации Apache2 и может быть легко расширена для конкретных нужд.