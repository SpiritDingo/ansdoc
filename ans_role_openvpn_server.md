Вот пример Ansible роли для установки и настройки OpenVPN сервера с использованием easy-rsa:

Структура роли:

```
roles/openvpn-server/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   ├── server.conf.j2
│   ├── client.conf.j2
│   ├── vars.j2
│   └── iptables-rules.j2
├── files
│   ├── generate-client.sh
│   └── revoke-client.sh
└── handlers
    └── main.yml
```

1. defaults/main.yml

```yaml
# Параметры OpenVPN сервера
openvpn_port: 1194
openvpn_proto: udp
openvpn_server_network: "10.8.0.0"
openvpn_server_netmask: "255.255.255.0"
openvpn_server_cidr: "10.8.0.0/24"
openvpn_dns_servers:
  - "8.8.8.8"
  - "8.8.4.4"

# Параметры сертификатов
openvpn_country: "RU"
openvpn_province: "Moscow"
openvpn_city: "Moscow"
openvpn_org: "MyCompany"
openvpn_email: "admin@example.com"
openvpn_ou: "IT"

# Ключи
openvpn_key_size: 2048
openvpn_dh_size: 2048
openvpn_ca_expire: 3650
openvpn_cert_expire: 365
openvpn_cert_serial: "01"

# Настройки безопасности
openvpn_cipher: "AES-256-CBC"
openvpn_auth: "SHA256"
openvpn_tls_version: "1.2"

# Список клиентов для создания
openvpn_clients:
  - client1
  - client2

# Сетевые настройки
openvpn_enable_ip_forward: true
openvpn_enable_nat: true
openvpn_public_interface: "eth0"
```

2. tasks/main.yml

```yaml
---
- name: Установка зависимостей
  apt:
    name:
      - openvpn
      - easy-rsa
      - iptables-persistent
    state: present
    update_cache: yes
  tags:
    - installation

- name: Создание директорий для OpenVPN
  file:
    path: "{{ item }}"
    state: directory
    mode: 0755
  loop:
    - /etc/openvpn
    - /etc/openvpn/easy-rsa
    - /etc/openvpn/clients
    - /etc/openvpn/ccd
  tags:
    - setup

- name: Копирование easy-rsa скриптов
  copy:
    src: /usr/share/easy-rsa/
    dest: /etc/openvpn/easy-rsa/
    mode: 0755
  tags:
    - certificates

- name: Создание файла переменных для easy-rsa
  template:
    src: vars.j2
    dest: /etc/openvpn/easy-rsa/vars
    mode: 0644
  tags:
    - certificates

- name: Инициализация PKI
  shell:
    cmd: |
      cd /etc/openvpn/easy-rsa
      ./easyrsa init-pki
    creates: /etc/openvpn/easy-rsa/pki/index.txt
  tags:
    - certificates

- name: Создание CA сертификата
  shell:
    cmd: |
      cd /etc/openvpn/easy-rsa
      ./easyrsa build-ca nopass
    creates: /etc/openvpn/easy-rsa/pki/ca.crt
  args:
    stdin: "{{ openvpn_country }}\n{{ openvpn_province }}\n{{ openvpn_city }}\n{{ openvpn_org }}\n{{ openvpn_ou }}\n{{ openvpn_email }}\n\n"
  tags:
    - certificates

- name: Генерация серверного сертификата
  shell:
    cmd: |
      cd /etc/openvpn/easy-rsa
      ./easyrsa gen-req server nopass
      ./easyrsa sign-req server server
    creates: /etc/openvpn/easy-rsa/pki/issued/server.crt
  tags:
    - certificates

- name: Генерация Diffie-Hellman параметров
  shell:
    cmd: |
      cd /etc/openvpn/easy-rsa
      ./easyrsa gen-dh
    creates: /etc/openvpn/easy-rsa/pki/dh.pem
    timeout: 300
  tags:
    - certificates

- name: Генерация TLS ключа
  command: openvpn --genkey --secret /etc/openvpn/easy-rsa/pki/ta.key
    creates: /etc/openvpn/easy-rsa/pki/ta.key
  tags:
    - certificates

- name: Копирование сертификатов и ключей в директорию OpenVPN
  copy:
    src: "/etc/openvpn/easy-rsa/pki/{{ item.src }}"
    dest: "/etc/openvpn/{{ item.dest }}"
    mode: "{{ item.mode | default('0644') }}"
    remote_src: yes
  loop:
    - { src: 'ca.crt', dest: 'ca.crt' }
    - { src: 'issued/server.crt', dest: 'server.crt' }
    - { src: 'private/server.key', dest: 'server.key', mode: '0600' }
    - { src: 'dh.pem', dest: 'dh.pem' }
    - { src: 'ta.key', dest: 'ta.key', mode: '0600' }
  tags:
    - certificates

- name: Создание конфигурации сервера
  template:
    src: server.conf.j2
    dest: /etc/openvpn/server.conf
    mode: 0644
  notify: restart openvpn
  tags:
    - configuration

- name: Включение IP forwarding
  sysctl:
    name: net.ipv4.ip_forward
    value: 1
    sysctl_set: yes
    state: present
    reload: yes
  when: openvpn_enable_ip_forward
  tags:
    - networking

- name: Настройка iptables правил
  template:
    src: iptables-rules.j2
    dest: /etc/iptables/rules.v4
    mode: 0644
  when: openvpn_enable_nat
  tags:
    - networking

- name: Создание клиентских сертификатов
  shell:
    cmd: |
      cd /etc/openvpn/easy-rsa
      ./easyrsa gen-req {{ item }} nopass
      ./easyrsa sign-req client {{ item }}
  loop: "{{ openvpn_clients }}"
  tags:
    - clients

- name: Копирование скриптов для управления клиентами
  copy:
    src: "{{ item }}"
    dest: "/usr/local/bin/{{ item }}"
    mode: 0755
  loop:
    - generate-client.sh
    - revoke-client.sh
  tags:
    - scripts

- name: Запуск и включение OpenVPN службы
  systemd:
    name: openvpn@server
    state: started
    enabled: yes
    daemon_reload: yes
  tags:
    - service
```

3. templates/server.conf.j2

```jinja2
port {{ openvpn_port }}
proto {{ openvpn_proto }}
dev tun

ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
dh /etc/openvpn/dh.pem
tls-auth /etc/openvpn/ta.key 0

server {{ openvpn_server_network }} {{ openvpn_server_netmask }}

ifconfig-pool-persist /var/log/openvpn/ipp.txt

push "redirect-gateway def1 bypass-dhcp"
{% for dns in openvpn_dns_servers %}
push "dhcp-option DNS {{ dns }}"
{% endfor %}

keepalive 10 120
cipher {{ openvpn_cipher }}
auth {{ openvpn_auth }}

user nobody
group nogroup
persist-key
persist-tun

status /var/log/openvpn/openvpn-status.log
log-append /var/log/openvpn/openvpn.log
verb 3

explicit-exit-notify 1

# TLS
tls-version-min {{ openvpn_tls_version }}
tls-cipher TLS-ECDHE-RSA-WITH-AES-256-GCM-SHA384

# Security
reneg-sec 3600

# Client config directory
client-config-dir /etc/openvpn/ccd
```

4. templates/vars.j2

```jinja2
set_var EASYRSA_REQ_COUNTRY    "{{ openvpn_country }}"
set_var EASYRSA_REQ_PROVINCE   "{{ openvpn_province }}"
set_var EASYRSA_REQ_CITY       "{{ openvpn_city }}"
set_var EASYRSA_REQ_ORG        "{{ openvpn_org }}"
set_var EASYRSA_REQ_EMAIL      "{{ openvpn_email }}"
set_var EASYRSA_REQ_OU         "{{ openvpn_ou }}"

set_var EASYRSA_KEY_SIZE       {{ openvpn_key_size }}
set_var EASYRSA_CA_EXPIRE      {{ openvpn_ca_expire }}
set_var EASYRSA_CERT_EXPIRE    {{ openvpn_cert_expire }}
set_var EASYRSA_CERT_SERIAL    {{ openvpn_cert_serial }}

set_var EASYRSA_DIGEST         "sha256"
```

5. templates/iptables-rules.j2

```jinja2
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s {{ openvpn_server_cidr }} -o {{ openvpn_public_interface }} -j MASQUERADE
COMMIT

*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -i {{ openvpn_public_interface }} -p {{ openvpn_proto }} --dport {{ openvpn_port }} -j ACCEPT
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i tun+ -j ACCEPT
-A FORWARD -i tun+ -o {{ openvpn_public_interface }} -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i {{ openvpn_public_interface }} -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT
COMMIT
```

6. files/generate-client.sh

```bash
#!/bin/bash

if [ -z "$1" ]; then
    echo "Usage: $0 <client-name>"
    exit 1
fi

CLIENT_NAME=$1
EASYRSA_DIR=/etc/openvpn/easy-rsa
CLIENT_DIR=/etc/openvpn/clients/$CLIENT_NAME

mkdir -p $CLIENT_DIR

cd $EASYRSA_DIR

# Генерация клиентского сертификата
./easyrsa gen-req $CLIENT_NAME nopass
./easyrsa sign-req client $CLIENT_NAME

# Создание клиентской конфигурации
cat > $CLIENT_DIR/$CLIENT_NAME.ovpn << EOF
client
dev tun
proto {{ openvpn_proto }}
remote {{ ansible_default_ipv4.address }} {{ openvpn_port }}
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher {{ openvpn_cipher }}
auth {{ openvpn_auth }}
verb 3
<ca>
$(cat /etc/openvpn/ca.crt)
</ca>
<cert>
$(cat $EASYRSA_DIR/pki/issued/$CLIENT_NAME.crt)
</cert>
<key>
$(cat $EASYRSA_DIR/pki/private/$CLIENT_NAME.key)
</key>
<tls-auth>
$(cat /etc/openvpn/ta.key)
</tls-auth>
EOF

echo "Client config created: $CLIENT_DIR/$CLIENT_NAME.ovpn"
```

7. handlers/main.yml

```yaml
---
- name: restart openvpn
  systemd:
    name: openvpn@server
    state: restarted
    daemon_reload: yes

- name: restart iptables
  service:
    name: netfilter-persistent
    state: restarted
```

8. Пример playbook (site.yml):

```yaml
---
- hosts: vpn_servers
  become: yes
  roles:
    - openvpn-server
  vars:
    openvpn_clients:
      - admin
      - user1
      - user2
```

Использование:

1. Создайте инвентарь с серверами:

```yaml
[vpn_servers]
vpn-server-01 ansible_host=192.168.1.10
```

1. Запустите playbook:

```bash
ansible-playbook -i inventory.yml site.yml
```

1. Для создания клиентского конфига на сервере:

```bash
/usr/local/bin/generate-client.sh client_name
```

Эта роль обеспечивает:

· Установку OpenVPN и зависимостей
· Настройку инфраструктуры PKI с easy-rsa
· Конфигурацию сервера OpenVPN
· Настройку сетевого взаимодействия (NAT, форвардинг)
· Создание клиентских сертификатов
· Безопасные настройки по умолчанию