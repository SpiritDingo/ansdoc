# Роль Ansible для установки и настройки BIND DNS сервера на Linux

Вот пример роли Ansible для установки и настройки BIND9 (Berkeley Internet Name Domain) DNS сервера на Linux (поддерживаются Debian/Ubuntu и RHEL/CentOS).

## Структура роли

```
roles/bind-dns-server/
├── defaults
│   └── main.yml       # Переменные по умолчанию
├── files
│   ├── named.conf.options  # Файл настроек BIND
│   └── db.example.com      # Пример зоны
├── handlers
│   └── main.yml       # Обработчики для перезагрузки службы
├── tasks
│   └── main.yml       # Основные задачи
└── templates
    ├── named.conf.local.j2  # Шаблон конфигурации локальных зон
    └── db.zone.j2          # Шаблон файла зоны
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки BIND
bind_package_name: "bind9"
bind_service_name: "bind9"
bind_config_dir: "/etc/bind"
bind_zones_dir: "{{ bind_config_dir }}/zones"
bind_user: "bind"
bind_listen_ipv4: "127.0.0.1"
bind_listen_ipv6: "::1"
bind_forwarders:
  - "8.8.8.8"
  - "8.8.4.4"
bind_allow_query: ["localhost"]
bind_recursion: "yes"
bind_dnssec_validation: "yes"

# Настройки зон
bind_zones:
  - name: "example.com"
    type: "master"
    soa: "ns1.example.com."
    soa_email: "admin.example.com"
    refresh: "86400"
    retry: "7200"
    expire: "3600000"
    ttl: "172800"
    nameservers:
      - "ns1.example.com."
    records:
      - name: "@"
        type: "A"
        value: "192.168.1.1"
      - name: "ns1"
        type: "A"
        value: "192.168.1.1"
      - name: "www"
        type: "A"
        value: "192.168.1.2"
```

### tasks/main.yml

```yaml
---
- name: Install BIND packages
  package:
    name: "{{ bind_package_name }}"
    state: present

- name: Create zones directory
  file:
    path: "{{ bind_zones_dir }}"
    state: directory
    owner: "{{ bind_user }}"
    group: "{{ bind_user }}"
    mode: "0755"

- name: Copy named.conf.options
  copy:
    src: "named.conf.options"
    dest: "{{ bind_config_dir }}/named.conf.options"
    owner: "{{ bind_user }}"
    group: "{{ bind_user }}"
    mode: "0644"
  notify: restart bind

- name: Configure local zones
  template:
    src: "named.conf.local.j2"
    dest: "{{ bind_config_dir }}/named.conf.local"
    owner: "{{ bind_user }}"
    group: "{{ bind_user }}"
    mode: "0644"
  notify: restart bind

- name: Create zone files
  template:
    src: "db.zone.j2"
    dest: "{{ bind_zones_dir }}/db.{{ item.name }}"
    owner: "{{ bind_user }}"
    group: "{{ bind_user }}"
    mode: "0644"
  with_items: "{{ bind_zones }}"
  notify: restart bind

- name: Enable and start BIND service
  service:
    name: "{{ bind_service_name }}"
    state: started
    enabled: yes
```

### handlers/main.yml

```yaml
---
- name: restart bind
  service:
    name: "{{ bind_service_name }}"
    state: restarted
```

### templates/named.conf.local.j2

```jinja2
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

{% for zone in bind_zones %}
zone "{{ zone.name }}" {
    type {{ zone.type }};
    file "{{ bind_zones_dir }}/db.{{ zone.name }}";
};
{% endfor %}
```

### templates/db.zone.j2

```jinja2
$TTL {{ item.ttl }}
@       IN      SOA     {{ item.soa }}. {{ item.soa_email }}. (
                      {{ ansible_date_time.epoch }} ; Serial
                      {{ item.refresh }}      ; Refresh
                      {{ item.retry }}       ; Retry
                      {{ item.expire }}      ; Expire
                      {{ item.ttl }} )    ; Negative Cache TTL
;

; Name servers
@       IN      NS      {{ item.nameservers[0] }}
{% for ns in item.nameservers[1:] %}
        IN      NS      {{ ns }}
{% endfor %}

; A records
{% for record in item.records %}
{% if record.type == "A" %}
{{ record.name }}       IN      A       {{ record.value }}
{% endif %}
{% endfor %}

; CNAME records
{% for record in item.records %}
{% if record.type == "CNAME" %}
{{ record.name }}       IN      CNAME       {{ record.value }}
{% endif %}
{% endfor %}
```

### files/named.conf.options

```
options {
    directory "/var/cache/bind";

    listen-on port 53 { {{ bind_listen_ipv4 }}; {{ bind_listen_ipv6 }}; };
    listen-on-v6 port 53 { {{ bind_listen_ipv6 }}; };

    allow-query { {% for query in bind_allow_query %}{{ query }}; {% endfor %} };

    recursion {{ bind_recursion }};
    allow-recursion { {% for query in bind_allow_query %}{{ query }}; {% endfor %} };

    forwarders {
        {% for forwarder in bind_forwarders %}{{ forwarder }};{% endfor %}
    };
    forward only;

    dnssec-validation {{ bind_dnssec_validation }};

    auth-nxdomain no;    # conform to RFC1035
};
```

## Пример playbook

```yaml
---
- name: Configure BIND DNS server
  hosts: dns_servers
  become: yes
  roles:
    - bind-dns-server
```

## Использование

1. Создайте структуру роли как показано выше
2. Настройте переменные в `defaults/main.yml` под ваши нужды
3. Создайте playbook, который будет применять эту роль
4. Запустите playbook: `ansible-playbook -i inventory.yml dns.yml`

## Дополнительные настройки

Для более сложных конфигураций вы можете добавить:
- Поддержку обратных зон (reverse DNS)
- Настройки DNSSEC
- ACL (Access Control Lists) для более тонкого управления доступом
- Интеграцию с динамическими DNS
- Логирование и мониторинг

Эта роль предоставляет базовую функциональность для настройки авторитативного DNS сервера с поддержкой прямых зон.