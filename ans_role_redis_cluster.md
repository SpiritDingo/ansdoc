Ниже представлена готовая Ansible роль для развёртывания кластера Redis 7.2.15 с тремя мастер-нодами и тремя репликами (по одной на каждый мастер). Роль устанавливает Redis из исходников, генерирует конфигурацию для каждого узла, запускает службы и инициализирует кластер.

Обзор роли

Роль выполняет:

· Установку Redis 7.2.15 из исходного кода на каждом хосте.
· Создание конфигурационного файла redis.conf для каждой ноды (мастер или реплика).
· Настройку systemd-юнита для запуска Redis.
· Запуск Redis на всех узлах.
· Инициализацию Redis Cluster с тремя мастерами и автоматическим назначением реплик (можно управлять вручную).
· Проверку состояния кластера.

Структура роли

```
roles/redis_cluster/
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
│   ├── redis.conf.j2
│   └── redis.service.j2
└── vars/
    └── main.yml
```

Переменные роли

defaults/main.yml

```yaml
---
# Версия Redis
redis_version: "7.2.15"
# Порт по умолчанию
redis_port: 6379
# Пользователь и группа
redis_user: redis
redis_group: redis
# Директории
redis_install_dir: "/opt/redis-{{ redis_version }}"
redis_config_dir: "/etc/redis"
redis_data_dir: "/var/lib/redis"
redis_log_dir: "/var/log/redis"
redis_pidfile: "/var/run/redis/redis.pid"
# Пароль для доступа (одинаковый для всех узлов)
redis_password: "your_secure_password"
# Параметры кластера
redis_cluster_enabled: true
redis_cluster_config_file: "nodes.conf"
redis_cluster_node_timeout: 5000
redis_appendonly: "yes"
redis_appendfsync: "everysec"
# Режим работы узла: master или replica
node_role: "master"          # переопределяется в inventory для реплик
master_of: ""                # имя мастер-ноды (для реплик)
```

vars/main.yml

```yaml
---
redis_url: "http://download.redis.io/releases/redis-{{ redis_version }}.tar.gz"
redis_bin_dir: "{{ redis_install_dir }}/bin"
redis_src_dir: "/usr/src/redis-{{ redis_version }}"
```

Шаблоны

templates/redis.conf.j2

```conf
# Базовые настройки
bind 0.0.0.0
protected-mode no
port {{ redis_port }}
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised systemd
pidfile {{ redis_pidfile }}
loglevel notice
logfile ""
databases 16
always-show-logo no

# Пароль
requirepass {{ redis_password }}
masterauth {{ redis_password }}

# Сохранение данных
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir {{ redis_data_dir }}

# Репликация (для реплик)
{% if node_role == "replica" %}
replicaof {{ master_ip }} {{ redis_port }}
{% endif %}
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
replica-priority 100

# Append Only Mode
appendonly {{ redis_appendonly }}
appendfsync {{ redis_appendfsync }}
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Кластер
cluster-enabled {{ redis_cluster_enabled }}
cluster-config-file {{ redis_cluster_config_file }}
cluster-node-timeout {{ redis_cluster_node_timeout }}
cluster-replica-validity-factor 10
cluster-migration-barrier 1
cluster-require-full-coverage yes
cluster-replica-no-failover no
cluster-allow-reads-when-down no
```

Замечание: для реплики требуется переменная master_ip — IP-адрес мастер-ноды. Она вычисляется в задачах через inventory.

templates/redis.service.j2

```
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
User={{ redis_user }}
Group={{ redis_group }}
ExecStart={{ redis_bin_dir }}/redis-server {{ redis_config_dir }}/redis.conf
ExecStop={{ redis_bin_dir }}/redis-cli -a {{ redis_password }} -p {{ redis_port }} shutdown
Restart=always
LimitNOFILE=10032

[Install]
WantedBy=multi-user.target
```

Задачи (tasks/main.yml)

```yaml
---
- name: Установка зависимостей для сборки
  ansible.builtin.apt:
    name:
      - build-essential
      - tcl
      - wget
      - make
      - gcc
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Создание пользователя redis
  ansible.builtin.user:
    name: "{{ redis_user }}"
    group: "{{ redis_group }}"
    system: yes
    create_home: no
    shell: /usr/sbin/nologin

- name: Создание директорий
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: "{{ redis_user }}"
    group: "{{ redis_group }}"
    mode: 0750
  loop:
    - "{{ redis_config_dir }}"
    - "{{ redis_data_dir }}"
    - "{{ redis_log_dir }}"
    - "/var/run/redis"
    - "{{ redis_install_dir }}"

- name: Скачивание исходников Redis
  ansible.builtin.get_url:
    url: "{{ redis_url }}"
    dest: "/tmp/redis-{{ redis_version }}.tar.gz"
    mode: 0644

- name: Распаковка исходников
  ansible.builtin.unarchive:
    src: "/tmp/redis-{{ redis_version }}.tar.gz"
    dest: "/usr/src/"
    remote_src: yes
    creates: "/usr/src/redis-{{ redis_version }}/README.md"

- name: Компиляция Redis
  ansible.builtin.command:
    cmd: make -j{{ ansible_processor_vcpus | default(1) }}
    chdir: "/usr/src/redis-{{ redis_version }}"
  environment:
    BUILD_TLS: "yes"   # если нужна поддержка TLS
  register: redis_make
  changed_when: "'error' not in redis_make.stderr"

- name: Установка бинарников
  ansible.builtin.command:
    cmd: make PREFIX={{ redis_install_dir }} install
    chdir: "/usr/src/redis-{{ redis_version }}"
  when: redis_make is succeeded

- name: Определение IP мастер-ноды для реплики (если применимо)
  ansible.builtin.set_fact:
    master_ip: "{{ hostvars[master_of]['ansible_default_ipv4']['address'] }}"
  when: node_role == "replica" and master_of is defined and master_of != ""

- name: Копирование конфигурации Redis
  ansible.builtin.template:
    src: redis.conf.j2
    dest: "{{ redis_config_dir }}/redis.conf"
    owner: "{{ redis_user }}"
    group: "{{ redis_group }}"
    mode: 0640
  notify: restart redis

- name: Копирование systemd unit
  ansible.builtin.template:
    src: redis.service.j2
    dest: /etc/systemd/system/redis.service
    owner: root
    group: root
    mode: 0644
  notify: restart redis

- name: Перечитывание systemd и запуск Redis
  ansible.builtin.systemd:
    name: redis
    enabled: yes
    state: started
    daemon_reload: yes

- name: Ожидание доступности порта Redis на всех узлах
  ansible.builtin.wait_for:
    host: "{{ ansible_default_ipv4.address }}"
    port: "{{ redis_port }}"
    state: started
    timeout: 30
  when: not ansible_check_mode

# === Инициализация кластера (выполняется один раз) ===
- name: Создание кластера из 3 мастер-нод и 1 реплики на каждый мастер
  ansible.builtin.command:
    cmd: >
      {{ redis_bin_dir }}/redis-cli --cluster create
      {% for node in groups['redis_cluster'] %}
        {{ hostvars[node]['ansible_default_ipv4']['address'] }}:{{ redis_port }}
      {% endfor %}
      --cluster-replicas 1
      -a {{ redis_password }}
  run_once: true
  delegate_to: "{{ groups['redis_cluster'][0] }}"
  register: cluster_create
  failed_when: "'ERR' in cluster_create.stderr"
  changed_when: "'All 16384 slots covered' in cluster_create.stdout"
  when: groups['redis_cluster'] | length == 6

- name: Проверка состояния кластера
  ansible.builtin.command:
    cmd: "{{ redis_bin_dir }}/redis-cli -a {{ redis_password }} cluster info"
  delegate_to: "{{ groups['redis_cluster'][0] }}"
  register: cluster_info
  changed_when: false
  run_once: true
  when: cluster_create is succeeded

- name: Вывод краткой информации о кластере
  ansible.builtin.debug:
    var: cluster_info.stdout_lines
```

Обработчики (handlers/main.yml)

```yaml
---
- name: restart redis
  ansible.builtin.systemd:
    name: redis
    state: restarted
    daemon_reload: yes
```

Инвентарь (пример)

Допустим, у нас есть 6 серверов (или 3 сервера с двумя экземплярами — но для простоты предположим 6 хостов). На каждом хосте один Redis-экземпляр.

```ini
[redis_cluster]
redis1 ansible_host=10.0.0.1 node_role=master
redis2 ansible_host=10.0.0.2 node_role=master
redis3 ansible_host=10.0.0.3 node_role=master
redis4 ansible_host=10.0.0.4 node_role=replica master_of=redis1
redis5 ansible_host=10.0.0.5 node_role=replica master_of=redis2
redis6 ansible_host=10.0.0.6 node_role=replica master_of=redis3
```

Важно: переменные node_role и master_of должны быть определены для каждого хоста (например, через host_vars или прямо в inventory). В шаблоне конфигурации для реплик используется replicaof, и роль автоматически подставит IP-адрес мастера из hostvars[master_of]['ansible_default_ipv4']['address'].

Если вы хотите использовать один и тот же redis_port для всех (обычно 6379), то команда создания кластера будет содержать 6 адресов с одинаковым портом.

Использование

1. Скопируйте роль в каталог roles/ вашего проекта.
2. Определите inventory с хостами кластера.
3. Запустите плейбук:

```yaml
---
- hosts: redis_cluster
  become: yes
  roles:
    - redis_cluster
```

После выполнения роль выведет информацию о состоянии кластера.

Примечания

· Точное соответствие реплик: текущая реализация использует --cluster-replicas 1, что автоматически распределяет реплики по мастерам. Если требуется жёсткая привязка конкретной реплики к конкретному мастеру, необходимо модифицировать блок инициализации: сначала создать кластер только из 3 мастеров, затем добавить реплики с указанием --cluster-master-id. Это можно сделать, получив ID мастеров через redis-cli cluster nodes.
· Безопасность: пароль передаётся в открытом виде в командах и конфигах. Для production рекомендуется использовать ansible-vault для шифрования переменной redis_password.
· Несколько экземпляров на хосте: для поддержки нескольких Redis-процессов на одной машине потребуется адаптировать роль (например, использовать цикл по redis_instances с разными портами и путями). В данной версии предполагается один экземпляр на хост.
· Firewall: убедитесь, что порты 6379 (и, если используется, порт кластера 16379) открыты между узлами.

Эта роль даёт полную автоматизацию развёртывания отказоустойчивого Redis Cluster с нужной версией.