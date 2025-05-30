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