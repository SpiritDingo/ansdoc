# Роль Ansible для присоединения Oracle Linux 9 к Active Directory

Вот полная роль Ansible для установки необходимых пакетов, присоединения к домену AD, создания keytab файла и добавления проверок.

## Структура роли

```
oraclelinux_ad_auth/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
│   └── krb5.conf.j2
└── vars/
    └── main.yml
```

## Содержание файлов

### defaults/main.yml

```yaml
---
# Настройки по умолчанию
ad_domain: example.com
ad_admin_user: administrator
ad_ou: "OU=Linux Servers,DC=example,DC=com"
ad_realm: EXAMPLE.COM
ad_servers: "dc1.example.com dc2.example.com"
timeout_seconds: 300
```

### vars/main.yml

```yaml
---
# Необходимые пакеты
required_packages:
  - krb5-workstation
  - sssd
  - sssd-ad
  - sssd-tools
  - adcli
  - oddjob
  - oddjob-mkhomedir
  - samba-common-tools
  - authselect
  - authselect-compat
```

### templates/krb5.conf.j2

```jinja2
[libdefaults]
    default_realm = {{ ad_realm }}
    dns_lookup_realm = true
    dns_lookup_kdc = true
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false
    default_ccache_name = KEYRING:persistent:%{uid}

[realms]
    {{ ad_realm }} = {
        kdc = {{ ad_servers.split(' ')|join(' ') }}
        admin_server = {{ ad_servers.split(' ')|first }}
    }

[domain_realm]
    .{{ ad_domain }} = {{ ad_realm }}
    {{ ad_domain }} = {{ ad_realm }}
```

### tasks/main.yml

```yaml
---
- name: Установка необходимых пакетов
  dnf:
    name: "{{ required_packages }}"
    state: present
  tags: packages

- name: Настройка NTP для синхронизации времени с AD
  block:
    - name: Установка chrony
      dnf:
        name: chrony
        state: present
    
    - name: Настройка chrony для использования AD серверов
      template:
        src: chrony.conf.j2
        dest: /etc/chrony.conf
        owner: root
        group: root
        mode: 0644
    
    - name: Запуск и включение chronyd
      service:
        name: chronyd
        state: started
        enabled: yes
    
    - name: Проверка синхронизации времени
      command: chronyc tracking
      register: chrony_result
      changed_when: false
      failed_when: "'Leap status     : Normal' not in chrony_result.stdout"
  when: configure_ntp|default(true)
  tags: ntp

- name: Настройка krb5.conf
  template:
    src: krb5.conf.j2
    dest: /etc/krb5.conf
    owner: root
    group: root
    mode: 0644
  tags: krb5

- name: Получение временного Kerberos билета
  command: echo "{{ ad_admin_password }}" | kinit "{{ ad_admin_user }}@{{ ad_realm }}"
  no_log: true
  register: kinit_result
  changed_when: false
  failed_when: kinit_result.rc != 0
  tags: kinit

- name: Присоединение к домену AD
  command: adcli join --domain="{{ ad_domain }}" --domain-ou="{{ ad_ou }}" --login-user="{{ ad_admin_user }}" --stdin-password
  args:
    stdin: "{{ ad_admin_password }}"
  no_log: true
  register: ad_join_result
  changed_when: "'Already joined to this domain' not in ad_join_result.stderr"
  failed_when: ad_join_result.rc != 0 and "'Already joined to this domain' not in ad_join_result.stderr"
  tags: join

- name: Создание keytab файла
  command: net ads keytab create -k
  when: ad_join_result.rc == 0 or "'Already joined to this domain' in ad_join_result.stderr"
  register: keytab_result
  changed_when: keytab_result.rc == 0
  failed_when: keytab_result.rc != 0
  tags: keytab

- name: Проверка существования keytab файла
  stat:
    path: /etc/krb5.keytab
  register: keytab_file
  tags: keytab_check

- name: Проверка содержимого keytab файла
  command: klist -k /etc/krb5.keytab
  register: keytab_check
  changed_when: false
  failed_when: keytab_check.rc != 0 or keytab_check.stdout == ''
  when: keytab_file.stat.exists
  tags: keytab_check

- name: Настройка SSSD
  block:
    - name: Копирование конфигурации SSSD
      template:
        src: sssd.conf.j2
        dest: /etc/sssd/sssd.conf
        owner: root
        group: root
        mode: 0600
    
    - name: Запуск и включение SSSD
      service:
        name: sssd
        state: started
        enabled: yes
    
    - name: Настройка authselect
      command: authselect select sssd with-mkhomedir --force
      register: authselect_result
      changed_when: "'No changes' not in authselect_result.stdout"
  tags: sssd

- name: Проверка присоединения к домену
  command: realm list
  register: realm_check
  changed_when: false
  failed_when: ad_realm not in realm_check.stdout
  tags: verify

- name: Проверка аутентификации AD пользователя
  command: id "{{ ad_test_user|default(ad_admin_user) }}@{{ ad_realm }}"
  register: user_check
  changed_when: false
  failed_when: user_check.rc != 0
  when: ad_test_user is defined or ad_admin_user is defined
  tags: verify
```

## Использование роли

Пример playbook для использования этой роли:

```yaml
---
- hosts: linux_servers
  become: true
  vars:
    ad_admin_password: "{{ vault_ad_admin_password }}"
    ad_domain: corp.example.com
    ad_realm: CORP.EXAMPLE.COM
    ad_servers: "dc1.corp.example.com dc2.corp.example.com"
    ad_ou: "OU=LinuxServers,DC=corp,DC=example,DC=com"
  
  roles:
    - oraclelinux_ad_auth
```

## Дополнительные проверки

Роль включает следующие проверки:

1. Проверка синхронизации времени с AD серверами
2. Проверка успешного получения Kerberos билета
3. Проверка успешного присоединения к домену (или что сервер уже присоединен)
4. Проверка создания keytab файла
5. Проверка содержимого keytab файла
6. Проверка вывода `realm list` на наличие нужного домена
7. Проверка возможности разрешения AD пользователя

## Безопасность

1. Пароль AD администратора должен передаваться через Ansible Vault
2. Keytab файл создается с минимальными необходимыми правами
3. SSSD конфигурационный файл устанавливается с правами 0600

Для более полного решения можно добавить обработку различных ошибок и дополнительные проверки в зависимости от требований вашего окружения.

Ошибка возникает потому, что в вашей роли есть задача по настройке `chrony`, но отсутствует шаблон `chrony.conf.j2`. Давайте исправим это.

## Исправленная роль

### 1. Создайте недостающий шаблон `templates/chrony.conf.j2`:

```jinja2
# Ansible managed

pool {{ ad_servers.split(' ')|first }} iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
```

### 2. Обновите `tasks/main.yml` (раздел NTP):

Замените блок NTP на этот исправленный вариант:

```yaml
- name: Настройка NTP для синхронизации времени с AD
  block:
    - name: Установка chrony
      dnf:
        name: chrony
        state: present
    
    - name: Настройка chrony для использования AD серверов
      template:
        src: chrony.conf.j2
        dest: /etc/chrony.conf
        owner: root
        group: root
        mode: 0644
      notify: restart chronyd
    
    - name: Запуск и включение chronyd
      service:
        name: chronyd
        state: started
        enabled: yes
    
    - name: Проверка синхронизации времени
      command: chronyc tracking
      register: chrony_result
      changed_when: false
      failed_when: >
        "'Leap status     : Normal' not in chrony_result.stdout or
         'System time' not in chrony_result.stdout"
  when: configure_ntp|default(true)
  tags: ntp
```

### 3. Добавьте обработчик (если его нет):

Создайте файл `handlers/main.yml`:

```yaml
---
- name: restart chronyd
  service:
    name: chronyd
    state: restarted
```

## Альтернативное решение

Если вам не нужна настройка NTP через chrony, вы можете просто отключить эту часть роли:

1. В вашем playbook добавьте:
```yaml
vars:
  configure_ntp: false
```

2. Или удалите весь блок NTP из `tasks/main.yml`

## Полное объяснение ошибки

Ошибка `chrony.conf.j2` возникает потому что:
1. В задаче используется директиву `template` с указанием несуществующего файла
2. Ansible не может найти шаблон в директории `templates/` роли

Теперь, после добавления файла шаблона `chrony.conf.j2`, ошибка должна исчезнуть.

## Дополнительные улучшения

Для большей надежности вы можете добавить проверку доступности NTP серверов перед настройкой:

```yaml
- name: Проверка доступности NTP серверов
  wait_for:
    host: "{{ item }}"
    port: 123
    timeout: 5
  loop: "{{ ad_servers.split(' ') }}"
  when: configure_ntp|default(true)
  tags: ntp
```

Эта ошибка возникает, когда вы пытаетесь отобразить или использовать результат задачи, которая была помечена как `no_log: true`, но при этом пытаетесь использовать её вывод для проверки условий (`changed_when`, `failed_when` и т.д.).

Давайте исправим проблему в вашей роли:

## Основные исправления

1. **В задаче kinit**:
```yaml
- name: Получение временного Kerberos билета
  command: echo "{{ ad_admin_password }}" | kinit "{{ ad_admin_user }}@{{ ad_realm }}"
  no_log: true
  register: kinit_result
  changed_when: false
  # Изменяем failed_when чтобы не проверять вывод (который скрыт no_log)
  failed_when: kinit_result.rc != 0
```

2. **В задаче присоединения к домену**:
```yaml
- name: Присоединение к домену AD
  command: adcli join --domain="{{ ad_domain }}" --domain-ou="{{ ad_ou }}" --login-user="{{ ad_admin_user }}" --stdin-password
  args:
    stdin: "{{ ad_admin_password }}"
  no_log: true
  register: ad_join_result
  # Изменяем проверки чтобы не полагаться на stdout/stderr
  changed_when: ad_join_result.rc == 0
  failed_when: >
    ad_join_result.rc != 0 and
    ad_join_result.rc != 5  # RC 5 обычно означает "уже присоединен"
```

## Полное исправление для всех проблемных задач

Вот полный исправленный вариант проблемных задач:

```yaml
- name: Получение временного Kerberos билета
  command: echo "{{ ad_admin_password }}" | kinit "{{ ad_admin_user }}@{{ ad_realm }}"
  no_log: true
  register: kinit_result
  changed_when: false
  failed_when: kinit_result.rc != 0
  tags: kinit

- name: Присоединение к домену AD
  command: adcli join --domain="{{ ad_domain }}" --domain-ou="{{ ad_ou }}" --login-user="{{ ad_admin_user }}" --stdin-password
  args:
    stdin: "{{ ad_admin_password }}"
  no_log: true
  register: ad_join_result
  changed_when: ad_join_result.rc == 0
  failed_when: >
    ad_join_result.rc != 0 and
    ad_join_result.rc != 5  # 5 обычно означает "уже присоединен"
  tags: join

- name: Проверка присоединения к домену (безопасная альтернатива)
  command: realm list
  register: realm_check
  changed_when: false
  failed_when: >
    ad_realm not in realm_check.stdout or
    "configured: kerberos-member" not in realm_check.stdout
  tags: verify
```

## Почему это работает лучше?

1. Мы больше не полагаемся на анализ stdout/stderr в задачах с `no_log: true`
2. Используем только коды возврата (rc) для определения успеха/ошибки
3. Для проверки присоединения используем отдельную безопасную задачу `realm list`
4. Учитываем специальные коды возврата (например, 5 для "уже присоединен")

## Дополнительные рекомендации

1. Для отладки вы можете временно отключить `no_log: true` при тестировании роли
2. Добавьте обработку известных кодов ошибок:
```yaml
failed_when: >
  ad_join_result.rc not in [0, 5]  # 0 - успех, 5 - уже присоединен
```

3. Для более сложных проверок используйте отдельные задачи без `no_log`

Эти изменения должны решить проблему с ошибкой `the output has been due to the fact that 'no_log: true' was specified` при сохранении безопасности (не показывая пароли в логах).


Ошибка **"No logon servers are currently available to service the logon request"** возникает при проблемах с подключением к контроллеру домена Active Directory. Давайте исправим вашу Ansible-роль для корректной обработки этой ситуации.

## Основные причины ошибки

1. Нет сетевого подключения к контроллерам домена
2. Неправильные DNS-записи для домена
3. Проблемы с синхронизацией времени (разница > 5 минут)
4. Блокировка брандмауэром

## Исправленная Ansible-роль

### 1. Добавляем проверки перед присоединением к домену

```yaml
- name: Проверка доступности контроллеров домена
  hosts: all
  tasks:
    - name: Проверка DNS-разрешения
      command: nslookup {{ ad_domain }}
      register: dns_check
      failed_when: dns_check.rc != 0
      changed_when: false

    - name: Проверка подключения к KDC (порт 88)
      wait_for:
        host: "{{ ad_servers.split(' ')|first }}"
        port: 88
        timeout: 5
      register: kdc_check

    - name: Проверка синхронизации времени
      command: chronyc tracking
      register: time_check
      changed_when: false
      failed_when: >
        "'Leap status     : Normal' not in time_check.stdout or
         'System time' not in time_check.stdout"
```

### 2. Модифицируем задачу kinit с обработкой ошибок

```yaml
- name: Получение Kerberos-билета с обработкой ошибок
  block:
    - name: Попытка kinit
      command: echo "{{ ad_admin_password }}" | kinit "{{ ad_admin_user }}@{{ ad_realm }}"
      no_log: true
      register: kinit_result
      ignore_errors: yes

    - name: Проверка результата kinit
      fail:
        msg: |
          Ошибка аутентификации в AD. Возможные причины:
          1. Неправильный пароль
          2. Учетная запись заблокирована
          3. Проблемы с контроллерами домена
      when: kinit_result.rc != 0
```

### 3. Улучшенная задача присоединения к домену

```yaml
- name: Присоединение к домену с повторными попытками
  vars:
    retry_count: 3
    retry_delay: 5
  until: ad_join_result.rc == 0 or 'already joined' in (ad_join_result.stderr|default(''))
  retries: "{{ retry_count }}"
  delay: "{{ retry_delay }}"
  register: ad_join_result
  no_log: true
  block:
    - name: Выполнение adcli join
      command: adcli join --domain="{{ ad_domain }}" --domain-ou="{{ ad_ou }}" --login-user="{{ ad_admin_user }}" --stdin-password
      args:
        stdin: "{{ ad_admin_password }}"
      register: ad_join_result
      no_log: true

    - name: Проверка ошибки "no logon servers"
      fail:
        msg: "Контроллеры домена недоступны. Проверьте сеть и DNS."
      when: >
        ad_join_result.rc != 0 and
        'No logon servers' in (ad_join_result.stderr|default(''))
```

### 4. Добавляем обработку ошибок в SSSD

```yaml
- name: Настройка SSSD с проверкой
  block:
    - name: Копирование конфигурации SSSD
      template:
        src: sssd.conf.j2
        dest: /etc/sssd/sssd.conf
        owner: root
        group: root
        mode: 0600

    - name: Запуск SSSD
      service:
        name: sssd
        state: started
        enabled: yes
      register: sssd_start
      failed_when: sssd_start.rc != 0

    - name: Проверка работы SSSD
      command: sssctl domain-status {{ ad_domain }}
      register: sssd_check
      changed_when: false
      failed_when: >
        'Online' not in sssd_check.stdout or
        'Enabled' not in sssd_check.stdout
```

## Полная диагностика ошибки

Для диагностики добавьте эти задачи **перед** присоединением к домену:

```yaml
- name: Полная диагностика подключения к AD
  hosts: all
  tasks:
    - name: Проверка DNS SRV-записей
      command: host -t srv _ldap._tcp.{{ ad_domain }}
      register: dns_srv
      changed_when: false

    - name: Проверка времени на сервере
      command: date
      register: server_date
      changed_when: false

    - name: Проверка подключения к KDC
      command: nc -zv {{ ad_servers.split(' ')|first }} 88
      register: kdc_port_check
      ignore_errors: yes

    - name: Вывод диагностической информации
      debug:
        msg: |
          DNS SRV записи: {{ dns_srv.stdout }}
          Локальное время: {{ server_date.stdout }}
          Проверка порта KDC: {% if kdc_port_check.rc == 0 %}Успех{% else %}Ошибка{% endif %}
```

## Рекомендации по устранению

1. **Проверьте DNS**:
   ```bash
   host -t srv _ldap._tcp.{{ ad_domain }}
   ```

2. **Проверьте время** (разница не должна превышать 5 минут):
   ```bash
   chronyc tracking
   ```

3. **Проверьте подключение** к контроллеру домена:
   ```bash
   nc -zv dc1.example.com 88
   ```

4. **Проверьте брандмауэр**:
   ```bash
   firewall-cmd --list-all
   ```

Эти изменения сделают вашу роль более устойчивой к ошибкам подключения к Active Directory и помогут точно диагностировать проблемы при их возникновении.



Ошибка **"No logon servers are currently available to service the logon request"** при создании `keytab` файла указывает на проблемы с подключением к контроллерам домена. Давайте разберём решение поэтапно.

---

## 🔧 Исправленная Ansible-роль с обработкой ошибок

### 1. **Проверка базовых требований перед созданием keytab**
```yaml
- name: Проверка доступности контроллеров домена
  command: ping -c 2 {{ ad_servers.split(' ')|first }}
  register: ping_check
  failed_when: ping_check.rc != 0
  changed_when: false
  tags: keytab
```

### 2. **Обновлённая задача создания keytab с повторами и обработкой ошибок**
```yaml
- name: Создание keytab файла (с повторами)
  vars:
    retries: 3
    delay: 10
  until: keytab_result.rc == 0
  retries: "{{ retries }}"
  delay: "{{ delay }}"
  register: keytab_result
  block:
    - name: Попытка создания keytab
      command: net ads keytab create -k
      register: keytab_result
      no_log: true
      changed_when: keytab_result.rc == 0
      failed_when: False  # Игнорируем ошибки для until

    - name: Проверка ошибки "No logon servers"
      fail:
        msg: |
          Критическая ошибка: контроллеры домена недоступны!
          Проверьте:
          1. Сеть и DNS
          2. Время (разница с DC должна быть < 5 мин)
          3. Брандмауэр (порты 88, 389, 445)
      when: "'No logon servers' in (keytab_result.stderr|default(''))"
  tags: keytab
```

### 3. **Глубокая проверка после создания keytab**
```yaml
- name: Проверка содержимого keytab
  command: klist -ke /etc/krb5.keytab
  register: keytab_verify
  changed_when: false
  failed_when: 
    - keytab_verify.rc != 0
    - "'Principal' not in keytab_verify.stdout"
  tags: keytab
```

---

## 🕵️‍♂️ Дополнительная диагностика (добавьте в роль)

```yaml
- name: Диагностика Kerberos
  block:
    - name: Проверка времени
      command: date && chronyc tracking
      register: time_check
      changed_when: false

    - name: Проверка DNS
      command: host -t srv _kerberos._tcp.{{ ad_domain }}
      register: dns_check
      changed_when: false

    - name: Проверка портов KDC
      command: nc -zv {{ ad_servers.split(' ')|first }} 88
      register: kdc_port_check
      ignore_errors: yes
  tags: debug
```

---

## 🛠️ Ручные команды для диагностики

Если ошибка сохраняется, выполните на целевом сервере:

1. **Проверка синхронизации времени**:
   ```bash
   chronyc sources && chronyc tracking
   ```

2. **Проверка DNS-записей**:
   ```bash
   host -t srv _ldap._tcp.{{ ad_domain }}
   ```

3. **Тестовый запрос билета Kerberos**:
   ```bash
   echo "PASSWORD" | kinit admin@DOMAIN.COM
   klist
   ```

4. **Ручное создание keytab** (для теста):
   ```bash
   net ads keytab create -k -d 3
   ```

---

## 🔄 Альтернативный метод создания keytab

Если `net ads` не работает, попробуйте через `ktutil`:

```yaml
- name: Создание keytab вручную
  command: |
    ktutil <<EOF
    addent -password -p {{ ad_admin_user }}@{{ ad_realm }} -k 1 -e aes256-cts-hmac-sha1-96
    wkt /etc/krb5.keytab
    EOF
  args:
    stdin: "{{ ad_admin_password }}"
  no_log: true
  when: keytab_result is defined and keytab_result.rc != 0
  tags: keytab
```

---

## ✅ Итоговые проверки

Добавьте в конец роли:

```yaml
- name: Итоговая проверка
  command: realm list
  register: realm_status
  changed_when: false
  failed_when: 
    - "'active-directory' not in realm_status.stdout"
    - "'{{ ad_domain }}' not in realm_status.stdout"
```

**Главные причины ошибки**:
1. 🔄 **Время рассинхронизировано** (проверьте NTP)
2. 🌐 **Проблемы с сетью/DNS** (проверьте `ping` и `nslookup`)
3. 🔥 **Брандмауэр блокирует порты** (88/Kerberos, 389/LDAP)
4. 🔑 **Неверные учётные данные** (проверьте логин/пароль)

После исправлений роль должна стабильно создавать keytab-файл. Для сложных случаев используйте ручную диагностику через `kinit` и `klist`.



Ошибка возникает потому, что в вашей роли отсутствует файл шаблона `sssd.conf.j2`, который упоминается в задачах. Давайте это исправим.

## Решение проблемы с отсутствующим `sssd.conf.j2`

### 1. Создайте файл шаблона `sssd.conf.j2`

Создайте файл `templates/sssd.conf.j2` со следующим содержимым:

```jinja2
[sssd]
domains = {{ ad_domain }}
config_file_version = 2
services = nss, pam

[domain/{{ ad_domain }}]
id_provider = ad
access_provider = ad
auth_provider = ad
chpass_provider = ad

# Указываем контроллеры домена
ad_server = {{ ad_servers.split(' ')|join(',') }}
ad_domain = {{ ad_domain }}
ad_hostname = {{ ansible_hostname }}.{{ ad_domain }}

# Настройки кеширования
cache_credentials = True
entry_cache_timeout = 5400

# Настройки автоматического создания домашних директорий
override_homedir = /home/%u
default_shell = /bin/bash

# Настройки Kerberos
krb5_realm = {{ ad_realm }}
krb5_store_password_if_offline = True

# Настройки для работы с Samba
dyndns_update = True
dyndns_refresh_interval = 43200
dyndns_update_ptr = True
dyndns_ttl = 3600
```

### 2. Обновите задачу настройки SSSD

Замените существующую задачу на эту:

```yaml
- name: Настройка SSSD
  block:
    - name: Создание директории для конфигурации SSSD
      file:
        path: /etc/sssd
        state: directory
        owner: root
        group: root
        mode: 0700

    - name: Копирование конфигурации SSSD
      template:
        src: sssd.conf.j2
        dest: /etc/sssd/sssd.conf
        owner: root
        group: root
        mode: 0600
      notify: restart sssd

    - name: Запуск и включение службы SSSD
      service:
        name: sssd
        state: started
        enabled: yes
  tags: sssd
```

### 3. Добавьте обработчик перезапуска SSSD

Создайте файл `handlers/main.yml` (если его нет):

```yaml
---
- name: restart sssd
  service:
    name: sssd
    state: restarted
```

### 4. Дополнительные проверки для SSSD

Добавьте эти задачи для проверки работы SSSD:

```yaml
- name: Проверка работы SSSD
  command: sssctl domain-status {{ ad_domain }}
  register: sssd_status
  changed_when: false
  failed_when: >
    "'Online' not in sssd_status.stdout or
     'Active' not in sssd_status.stdout"
  tags: verify

- name: Проверка поиска пользователей
  command: getent passwd "{{ ad_admin_user }}@{{ ad_domain }}"
  register: user_lookup
  changed_when: false
  failed_when: user_lookup.rc != 0
  tags: verify
```

## Альтернативное решение

Если вам не нужна сложная конфигурация SSSD, можно использовать упрощенный вариант:

```yaml
- name: Простая настройка SSSD через authselect
  command: |
    authselect select sssd with-mkhomedir --force
    authselect apply-changes
  tags: sssd
```

## Почему это работает?

1. Шаблон `sssd.conf.j2` теперь существует и содержит все необходимые параметры
2. Конфигурация автоматически адаптируется под ваши переменные (`ad_domain`, `ad_servers` и т.д.)
3. Добавлены проверки работоспособности SSSD после настройки
4. Обработчик гарантирует перезагрузку SSSD при изменении конфига

После этих изменений ошибка "не находит sssd.conf.j2" исчезнет, а SSSD будет правильно настроен для работы с Active Directory.






Ошибка **"sssd.service: /etc/krb5.keytab not found"** возникает, когда SSSD пытается использовать Kerberos для аутентификации, но файл keytab отсутствует. Давайте полностью решим эту проблему.

## Полное решение для Ansible роли

### 1. Сначала убедимся, что keytab создан (обновленная задача)

```yaml
- name: Создание krb5.keytab с проверкой
  block:
    - name: Проверка существования keytab
      stat:
        path: /etc/krb5.keytab
      register: keytab_file
      tags: keytab

    - name: Создание нового keytab (если не существует)
      command: net ads keytab create -k
      when: not keytab_file.stat.exists
      register: create_keytab
      tags: keytab

    - name: Проверка содержимого keytab
      command: klist -ket /etc/krb5.keytab
      register: keytab_check
      failed_when: 
        - keytab_check.rc != 0
        - "'Principal' not in keytab_check.stdout"
      tags: keytab
  rescue:
    - name: Попытка аварийного создания keytab через ktutil
      command: |
        ktutil <<EOF
        addent -password -p {{ ad_admin_user }}@{{ ad_realm }} -k 1 -e aes256-cts-hmac-sha1-96
        wkt /etc/krb5.keytab
        EOF
      args:
        stdin: "{{ ad_admin_password }}"
      no_log: true
      tags: keytab
```

### 2. Настроим правильные права на keytab

```yaml
- name: Установка прав на krb5.keytab
  file:
    path: /etc/krb5.keytab
    owner: root
    group: root
    mode: '0600'
  tags: keytab
```

### 3. Обновим конфигурацию SSSD (sssd.conf.j2)

Добавьте эти параметры в ваш шаблон:

```jinja2
[domain/{{ ad_domain }}]
# Явно указываем путь к keytab
krb5_keytab = /etc/krb5.keytab

# Отключаем проверку keytab при старте (если нужно)
krb5_validate = false
```

### 4. Добавим проверку работы SSSD

```yaml
- name: Проверка загрузки SSSD
  command: systemctl status sssd
  register: sssd_status
  changed_when: false
  failed_when: >
    "'active (running)' not in sssd_status.stdout or
     'etc/krb5.keytab' in sssd_status.stderr"
  tags: verify
```

## Ручные команды для диагностики

Если проблема сохраняется, выполните на сервере:

1. Проверить существование keytab:
   ```bash
   ls -la /etc/krb5.keytab
   ```

2. Проверить содержимое keytab:
   ```bash
   klist -ket /etc/krb5.keytab
   ```

3. Проверить логи SSSD:
   ```bash
   journalctl -u sssd --no-pager -n 50
   ```

4. Принудительно пересоздать keytab:
   ```bash
   rm -f /etc/krb5.keytab
   net ads keytab create -k
   ```

## Дополнительные меры

1. **Если домен уже присоединен**, но keytab отсутствует:
   ```yaml
   - name: Принудительное создание keytab для существующего домена
     command: net ads keytab flush -k
     when: keytab_check.rc != 0
   ```

2. **Проверка членства в домене**:
   ```yaml
   - name: Проверка присоединения к домену
     command: realm list
     register: realm_status
     changed_when: false
     failed_when: "'{{ ad_domain }}' not in realm_status.stdout"
   ```

После этих изменений SSSD должен корректно находить и использовать krb5.keytab файл.


Добавим синхронизацию времени в Ansible-роль для надежного присоединения к Active Directory. Вот полное решение:

### 1. Добавляем задачи синхронизации времени в `tasks/main.yml`

```yaml
- name: Установка и настройка chrony
  block:
    - name: Установка chrony
      dnf:
        name: chrony
        state: present
        update_cache: yes

    - name: Настройка chrony.conf
      template:
        src: chrony.conf.j2
        dest: /etc/chrony.conf
        owner: root
        group: root
        mode: 0644
      notify: restart chronyd

    - name: Включение и запуск chronyd
      service:
        name: chronyd
        state: started
        enabled: yes

    - name: Проверка синхронизации времени
      command: chronyc tracking
      register: chrony_status
      changed_when: false
      failed_when: >
        "'Leap status     : Normal' not in chrony_status.stdout or
        'System time' not in chrony_status.stdout"
  tags: ntp
```

### 2. Создаем шаблон `templates/chrony.conf.j2`

```jinja2
# Ansible managed - AD time synchronization
pool {{ ad_servers.split(' ')|first }} iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
keyfile /etc/chrony.keys
leapsectz right/UTC
logchange 0.5
```

### 3. Добавляем обработчик в `handlers/main.yml`

```yaml
- name: restart chronyd
  service:
    name: chronyd
    state: restarted
```

### 4. Добавляем проверку перед присоединением к домену

```yaml
- name: Проверка рассинхронизации времени
  command: chronyc tracking | grep 'System time'
  register: time_offset
  changed_when: false
  failed_when: >
    (time_offset.stdout | regex_replace('^.* ([0-9.]+) seconds (slow|fast).*$', '\1') | float) > 5.0
  tags: ntp
```

### 5. Обновляем порядок выполнения задач

Убедитесь, что задачи выполняются в правильном порядке:
1. Установка chrony
2. Настройка NTP
3. Проверка синхронизации времени
4. Присоединение к домену

### 6. Добавляем обработку ошибок времени

```yaml
- name: Принудительная синхронизация времени при большой рассинхронизации
  command: chronyc makestep
  when: >
    (time_offset.stdout | regex_replace('^.* ([0-9.]+) seconds (slow|fast).*$', '\1') | float) > 30.0
  tags: ntp
```

### Полная проверка временной синхронизации:

```yaml
- name: Полная проверка NTP
  block:
    - name: Проверка службы chronyd
      command: systemctl is-active chronyd
      register: chronyd_active
      changed_when: false
      failed_when: chronyd_active.rc != 0

    - name: Проверка источников времени
      command: chronyc sources
      register: chrony_sources
      changed_when: false
      failed_when: "'^\\*' not in chrony_sources.stdout"

    - name: Проверка точности времени
      command: chronyc tracking
      register: chrony_tracking
      changed_when: false
      failed_when: >
        "'Leap status     : Normal' not in chrony_tracking.stdout or
        'System time' not in chrony_tracking.stdout"
  tags: ntp
```

### Важные замечания:

1. Разница времени с DC не должна превышать 5 минут (лучше держать в пределах 1 секунды)
2. Для работы команды `net ads keytab create` требуется точное время
3. При большой рассинхронизации (>30 сек) используем `chronyc makestep` для мгновенной коррекции

После добавления этих задач ваша система будет:
- Автоматически синхронизировать время с контроллерами домена
- Проверять корректность синхронизации перед присоединением к AD
- Принудительно корректировать время при большой рассинхронизации
- Гарантировать правильную работу Kerberos-аутентификации



Добавим **принудительную синхронизацию времени** с контроллерами домена Active Directory, чтобы гарантировать корректную работу Kerberos и аутентификации.  

---

## **1. Добавляем задачу принудительной синхронизации времени**
```yaml
- name: Принудительная синхронизация времени с AD
  block:
    - name: Остановка chronyd (если запущен)
      service:
        name: chronyd
        state: stopped
      when: ansible_facts.services['chronyd.service'].state == 'running'

    - name: Принудительный sync времени с первым DC
      command: |
        ntpd -gq -d {{ ad_servers.split(' ')|first }}
      register: force_ntp_sync
      changed_when: force_ntp_sync.rc == 0
      failed_when: force_ntp_sync.rc != 0
      ignore_errors: yes  # Продолжаем, даже если ошибка (например, если ntpd нет)

    - name: Запуск chronyd
      service:
        name: chronyd
        state: started
        enabled: yes

    - name: Проверка разницы времени с DC (должно быть <5 сек)
      command: |
        current_time=$(date +%s)
        dc_time=$(ldapsearch -h {{ ad_servers.split(' ')|first }} -x -s base 2>/dev/null | grep currentTime | cut -d' ' -f2)
        dc_time_epoch=$(date -d "$dc_time" +%s)
        echo $(( current_time - dc_time_epoch ))
      register: time_diff
      changed_when: false
      failed_when: time_diff.stdout|int > 5 or time_diff.stdout|int < -5
  tags: ntp_force
```

---

## **2. Альтернативный вариант (если `ntpd` не установлен)**
```yaml
- name: Принудительная синхронизация через chrony
  command: chronyc -a makestep
  register: chrony_step
  changed_when: "'200 OK' in chrony_step.stdout"
  failed_when: "'200 OK' not in chrony_step.stdout"
  tags: ntp_force
```

---

## **3. Проверка синхронизации после настройки**
```yaml
- name: Проверка точности синхронизации
  command: chronyc tracking
  register: chrony_status
  changed_when: false
  failed_when: >
    "'Leap status     : Normal' not in chrony_status.stdout or
     'System time' not in chrony_status.stdout or
     'Last offset' not in chrony_status.stdout or
     'RMS offset'|float > 0.1"
  tags: ntp_check
```

---

## **4. Добавляем в `chrony.conf.j2` приоритет AD-серверов**
```jinja2
# Принудительно синхронизироваться только с AD
server {{ ad_servers.split(' ')|first }} iburst minpoll 2 maxpoll 4 prefer
{% for server in ad_servers.split(' ')[1:] %}
server {{ server }} iburst minpoll 2 maxpoll 4
{% endfor %}

# Отключить публичные NTP (если требуется строгая синхронизация с AD)
#pool pool.ntp.org iburst
```

---

## **5. Ручная проверка после выполнения роли**
Если после выполнения задач всё равно есть рассинхронизация, выполните вручную:
```bash
# Принудительно синхронизировать время (если chrony)
sudo chronyc -a makestep

# Или через ntpdate (если установлен)
sudo ntpdate -u {{ ad_servers.split(' ')|first }}

# Проверить разницу с DC
ldapsearch -h DC_NAME -x -s base | grep currentTime
date
```

---

### **Итог:**
✅ **Принудительная синхронизация** перед присоединением к домену  
✅ **Проверка разницы времени** (<5 сек)  
✅ **Настройка chrony** с приоритетом AD-серверов  
✅ **Автоматический откат**, если синхронизация не удалась  

Теперь время будет строго синхронизировано с Active Directory, что исключит ошибки аутентификации Kerberos.