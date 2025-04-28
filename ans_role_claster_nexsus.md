# Ansible роль для установки и настройки кластера Nexus

Роль Ansible для развертывания и настройки кластера Sonatype Nexus Repository Manager может выглядеть следующим образом. Вот пример структуры и содержания такой роли.

## Структура роли

```
roles/nexus_cluster/
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   ├── common.yml
│   ├── configure.yml
│   ├── install.yml
│   ├── main.yml
│   └── setup_cluster.yml
├── templates
│   ├── nexus.properties.j2
│   ├── nexus-rc.sh.j2
│   └── nexus.vmoptions.j2
└── vars
    └── main.yml
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Версия Nexus
nexus_version: "3.58.1-02"
nexus_download_url: "https://download.sonatype.com/nexus/3/nexus-{{ nexus_version }}-unix.tar.gz"

# Параметры установки
nexus_install_dir: "/opt"
nexus_data_dir: "/nexus-data"
nexus_user: "nexus"
nexus_group: "nexus"

# Параметры кластера
nexus_cluster_enabled: true
nexus_cluster_nodes:
  - "node1.example.com"
  - "node2.example.com"
  - "node3.example.com"
nexus_cluster_name: "nexus-cluster"
nexus_hazelcast_port: 5701

# Параметры JVM
nexus_jvm_min_heap: "2G"
nexus_jvm_max_heap: "2G"
nexus_jvm_extra_opts: ""
```

### tasks/main.yml

```yaml
---
- name: Include common tasks
  include_tasks: common.yml

- name: Include installation tasks
  include_tasks: install.yml

- name: Include configuration tasks
  include_tasks: configure.yml
  when: nexus_cluster_enabled

- name: Include cluster setup tasks
  include_tasks: setup_cluster.yml
  when: nexus_cluster_enabled
```

### tasks/install.yml

```yaml
---
- name: Create nexus user and group
  user:
    name: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    system: yes
    create_home: no
    shell: /bin/bash

- name: Create installation directory
  file:
    path: "{{ nexus_install_dir }}"
    state: directory
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"

- name: Download Nexus
  get_url:
    url: "{{ nexus_download_url }}"
    dest: "/tmp/nexus-{{ nexus_version }}-unix.tar.gz"
    checksum: "sha256:<checksum_here>"

- name: Extract Nexus archive
  unarchive:
    src: "/tmp/nexus-{{ nexus_version }}-unix.tar.gz"
    dest: "{{ nexus_install_dir }}"
    remote_src: yes
    extra_opts: "--strip-components=1"
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"

- name: Create data directory
  file:
    path: "{{ nexus_data_dir }}"
    state: directory
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    mode: '0755'

- name: Configure Nexus runtime
  template:
    src: "nexus-rc.sh.j2"
    dest: "/etc/nexus-rc.sh"
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    mode: '0755'

- name: Configure JVM options
  template:
    src: "nexus.vmoptions.j2"
    dest: "{{ nexus_install_dir }}/bin/nexus.vmoptions"
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    mode: '0644'

- name: Create systemd service
  template:
    src: "nexus.service.j2"
    dest: "/etc/systemd/system/nexus.service"
    owner: root
    group: root
    mode: '0644'

- name: Enable and start Nexus service
  systemd:
    name: nexus
    enabled: yes
    state: started
    daemon_reload: yes
```

### tasks/setup_cluster.yml

```yaml
---
- name: Configure cluster properties
  template:
    src: "nexus.properties.j2"
    dest: "{{ nexus_data_dir }}/etc/nexus.properties"
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    mode: '0644'

- name: Ensure cluster configuration directory exists
  file:
    path: "{{ nexus_data_dir }}/etc/fabric"
    state: directory
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    mode: '0755'

- name: Configure Hazelcast cluster
  template:
    src: "hazelcast.xml.j2"
    dest: "{{ nexus_data_dir }}/etc/fabric/hazelcast.xml"
    owner: "{{ nexus_user }}"
    group: "{{ nexus_group }}"
    mode: '0644'

- name: Restart Nexus to apply cluster configuration
  systemd:
    name: nexus
    state: restarted
    daemon_reload: yes
```

### templates/nexus.properties.j2

```
# Jetty configuration
application-port=8081
application-host=0.0.0.0
nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-requestlog.xml
nexus-context-path=/

# Nexus clustering
nexus.clustered=true
nexus.licenseFile=/nexus-data/instance/nexus.lic

# Hazelcast configuration
nexus.hazelcast.discovery.isEnabled=true
nexus.hazelcast.discovery.aws=false
nexus.hazelcast.port={{ nexus_hazelcast_port }}
nexus.hazelcast.discovery.serviceName={{ nexus_cluster_name }}
```

### templates/hazelcast.xml.j2

```xml
<hazelcast xsi:schemaLocation="http://www.hazelcast.com/schema/config hazelcast-config-3.10.xsd"
           xmlns="http://www.hazelcast.com/schema/config"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <group>
        <name>{{ nexus_cluster_name }}</name>
    </group>
    <network>
        <port auto-increment="true">{{ nexus_hazelcast_port }}</port>
        <join>
            <multicast enabled="false"/>
            <tcp-ip enabled="true">
                {% for node in nexus_cluster_nodes %}
                <member>{{ node }}</member>
                {% endfor %}
            </tcp-ip>
        </join>
    </network>
</hazelcast>
```

## Использование роли

Пример playbook для использования этой роли:

```yaml
---
- name: Deploy Nexus Cluster
  hosts: nexus_nodes
  become: yes
  vars:
    nexus_cluster_nodes:
      - "nexus-node1.example.com"
      - "nexus-node2.example.com"
      - "nexus-node3.example.com"
    nexus_cluster_name: "production-nexus-cluster"
  
  roles:
    - nexus_cluster
```

## Дополнительные рекомендации

1. **Балансировка нагрузки**: Настройте балансировщик нагрузки (например, Nginx или HAProxy) перед кластером Nexus.
2. **Общее хранилище**: Для кластера Nexus требуется общее хранилище (NFS, S3 или другая распределенная файловая система) для директории данных.
3. **Лицензия**: Убедитесь, что у вас есть соответствующая лицензия Sonatype для работы в кластерном режиме.
4. **Безопасность**: Настройте межсерверное шифрование для Hazelcast и аутентификацию между узлами.
5. **Мониторинг**: Реализуйте мониторинг состояния кластера и отдельных узлов.

Эта роль предоставляет базовую настройку кластера Nexus. В зависимости от ваших требований, возможно, потребуется добавить дополнительные параметры конфигурации.