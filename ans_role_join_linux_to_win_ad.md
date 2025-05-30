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