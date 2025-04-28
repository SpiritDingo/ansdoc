# Ansible Role: Установка и настройка OpenVPN

Этот Ansible-роль выполняет установку и настройку OpenVPN сервера, а также предоставляет функционал для добавления и удаления клиентов.

## Функционал роли

1. Установка OpenVPN и зависимостей
2. Настройка сервера OpenVPN
3. Генерация необходимых сертификатов и ключей
4. Управление клиентами (добавление/удаление)
5. Настройка сетевых параметров и firewall

## Структура роли

```
openvpn/
├── defaults/
│   └── main.yml       # Основные переменные по умолчанию
├── files/
│   ├── server.conf.j2 # Шаблон конфигурации сервера
│   └── client.conf.j2 # Шаблон конфигурации клиента
├── tasks/
│   ├── main.yml       # Основные задачи
│   ├── install.yml    # Установка пакетов
│   ├── configure.yml  # Настройка сервера
│   ├── certs.yml      # Работа с сертификатами
│   ├── clients.yml    # Управление клиентами
│   └── firewall.yml   # Настройка firewall
├── templates/
│   └── ccd.j2         # Шаблон для клиентских конфигураций
└── vars/
    └── main.yml       # Основные переменные
```

## Использование

### Основные переменные (defaults/main.yml)

```yaml
# Основные параметры OpenVPN
openvpn_port: 1194
openvpn_proto: udp
openvpn_network: 10.8.0.0
openvpn_netmask: 255.255.255.0
openvpn_server: 10.8.0.1
openvpn_dns: 8.8.8.8

# Параметры сертификатов
openvpn_country: "US"
openvpn_province: "CA"
openvpn_city: "SanFrancisco"
openvpn_org: "OpenVPN"
openvpn_email: "admin@example.com"
openvpn_ou: "OpenVPN"

# Клиенты (добавляются автоматически при создании)
openvpn_clients: []
```

### Установка и настройка сервера

Пример playbook для установки сервера:

```yaml
- hosts: vpn_server
  become: yes
  roles:
    - role: openvpn
```

### Добавление клиента

Для добавления клиента используйте дополнительные переменные:

```yaml
- hosts: vpn_server
  become: yes
  vars:
    openvpn_client_name: "client1"
    openvpn_action: "add_client"
  roles:
    - role: openvpn
```

### Удаление клиента

Для удаления клиента:

```yaml
- hosts: vpn_server
  become: yes
  vars:
    openvpn_client_name: "client1"
    openvpn_action: "revoke_client"
  roles:
    - role: openvpn
```

## Реализация задач

### Основные задачи (tasks/main.yml)

```yaml
- name: Include installation tasks
  include_tasks: install.yml
  tags: install

- name: Include configuration tasks
  include_tasks: configure.yml
  tags: configure

- name: Include certificate tasks
  include_tasks: certs.yml
  tags: certs

- name: Include client management tasks
  include_tasks: clients.yml
  when: openvpn_action is defined
  tags: clients

- name: Include firewall tasks
  include_tasks: firewall.yml
  tags: firewall
```

### Управление клиентами (tasks/clients.yml)

```yaml
- name: Add OpenVPN client
  block:
    - name: Create client certificate and key
      command: "bash /etc/openvpn/easy-rsa/easyrsa build-client-full {{ openvpn_client_name }} nopass"
      args:
        chdir: /etc/openvpn/easy-rsa/
        creates: "/etc/openvpn/easy-rsa/pki/issued/{{ openvpn_client_name }}.crt"

    - name: Generate client configuration
      template:
        src: client.conf.j2
        dest: "/etc/openvpn/clients/{{ openvpn_client_name }}.ovpn"
        mode: 0644

    - name: Archive client config and keys
      archive:
        path:
          - "/etc/openvpn/easy-rsa/pki/private/{{ openvpn_client_name }}.key"
          - "/etc/openvpn/easy-rsa/pki/issued/{{ openvpn_client_name }}.crt"
          - "/etc/openvpn/clients/{{ openvpn_client_name }}.ovpn"
        dest: "/etc/openvpn/clients/{{ openvpn_client_name }}.zip"
        format: zip

    - name: Add client to list
      set_fact:
        openvpn_clients: "{{ openvpn_clients + [openvpn_client_name] }}"
  when: openvpn_action == 'add_client'

- name: Revoke OpenVPN client
  block:
    - name: Revoke client certificate
      command: "bash /etc/openvpn/easy-rsa/easyrsa revoke {{ openvpn_client_name }}"
      args:
        chdir: /etc/openvpn/easy-rsa/
      register: revoke_result
      failed_when: "'error' in revoke_result.stderr"

    - name: Update CRL
      command: "bash /etc/openvpn/easy-rsa/easyrsa gen-crl"
      args:
        chdir: /etc/openvpn/easy-rsa/

    - name: Remove client files
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - "/etc/openvpn/easy-rsa/pki/private/{{ openvpn_client_name }}.key"
        - "/etc/openvpn/easy-rsa/pki/issued/{{ openvpn_client_name }}.crt"
        - "/etc/openvpn/clients/{{ openvpn_client_name }}.ovpn"
        - "/etc/openvpn/clients/{{ openvpn_client_name }}.zip"

    - name: Remove client from list
      set_fact:
        openvpn_clients: "{{ openvpn_clients | reject('equalto', openvpn_client_name) | list }}"
  when: openvpn_action == 'revoke_client'
```

## Шаблоны конфигураций

### Шаблон сервера (files/server.conf.j2)

```
port {{ openvpn_port }}
proto {{ openvpn_proto }}
dev tun
ca /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server.crt
key /etc/openvpn/easy-rsa/pki/private/server.key
dh /etc/openvpn/easy-rsa/pki/dh.pem
server {{ openvpn_network }} {{ openvpn_netmask }}
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS {{ openvpn_dns }}"
keepalive 10 120
tls-auth /etc/openvpn/easy-rsa/pki/ta.key 0
cipher AES-256-CBC
persist-key
persist-tun
status /var/log/openvpn-status.log
verb 3
explicit-exit-notify 1
```

### Шаблон клиента (files/client.conf.j2)

```
client
dev tun
proto {{ openvpn_proto }}
remote {{ ansible_host }} {{ openvpn_port }}
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
verb 3
<ca>
{{ lookup('file', '/etc/openvpn/easy-rsa/pki/ca.crt') }}
</ca>
<cert>
{{ lookup('file', '/etc/openvpn/easy-rsa/pki/issued/' + openvpn_client_name + '.crt') }}
</cert>
<key>
{{ lookup('file', '/etc/openvpn/easy-rsa/pki/private/' + openvpn_client_name + '.key') }}
</key>
<tls-auth>
{{ lookup('file', '/etc/openvpn/easy-rsa/pki/ta.key') }}
</tls-auth>
key-direction 1
```

## Дополнительные возможности

1. **Автоматическое обновление CRL** при отзыве клиентского сертификата
2. **Архивация конфигураций** клиентов в zip-файлы для удобного скачивания
3. **Гибкая настройка параметров** VPN через переменные
4. **Поддержка разных протоколов** (UDP/TCP)

Для использования роли необходимо:
1. Скопировать роль в директорию roles/
2. Настроить переменные под свои нужды
3. Запустить playbook с нужными параметрами