# Универсальная Ansible роль для установки и настройки Minikube на Oracle Linux 9 и Ubuntu

Вот универсальная Ansible роль для установки Minikube на обе системы (Oracle Linux 9 и Ubuntu).

## Структура роли

```
roles/minikube/
├── defaults
│   └── main.yml
├── tasks
│   ├── main.yml
│   ├── oraclelinux.yml
│   └── ubuntu.yml
├── templates
│   └── minikube.service.j2
└── vars
    ├── main.yml
    ├── oraclelinux.yml
    └── ubuntu.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Минимальные версии для установки
minikube_version: "latest"
kubectl_version: "latest"
docker_version: "latest"

# Настройки Minikube
minikube_cpus: 2
minikube_memory: 4096
minikube_disk_size: "20g"
minikube_driver: "docker"  # или "kvm2", "virtualbox" и т.д.

# Настройки пользователя
minikube_user: "{{ ansible_user_id }}"
minikube_group: "{{ ansible_user_gid }}"
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_distribution|lower }}.yml"

- name: Include OS-specific tasks
  include: "{{ ansible_distribution|lower }}.yml"
  
- name: Install dependencies
  package:
    name: "{{ minikube_dependencies }}"
    state: present

- name: Download and install kubectl
  get_url:
    url: "https://storage.googleapis.com/kubernetes-release/release/{{ kubectl_version }}/bin/linux/amd64/kubectl"
    dest: /usr/local/bin/kubectl
    mode: '0755'
  when: kubectl_version != "latest"

- name: Install latest kubectl
  shell: |
    curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod +x kubectl
    mv kubectl /usr/local/bin/
  when: kubectl_version == "latest"

- name: Download and install minikube
  get_url:
    url: "https://storage.googleapis.com/minikube/releases/{{ minikube_version }}/minikube-linux-amd64"
    dest: /usr/local/bin/minikube
    mode: '0755'
  when: minikube_version != "latest"

- name: Install latest minikube
  shell: |
    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    chmod +x minikube
    mv minikube /usr/local/bin/
  when: minikube_version == "latest"

- name: Create minikube systemd service
  template:
    src: minikube.service.j2
    dest: /etc/systemd/system/minikube.service
    owner: root
    group: root
    mode: '0644'

- name: Add user to docker group
  user:
    name: "{{ minikube_user }}"
    groups: docker
    append: yes

- name: Reload systemd
  systemd:
    daemon_reload: yes

- name: Enable and start docker service
  service:
    name: docker
    state: started
    enabled: yes

- name: Start minikube
  command: "minikube start --driver={{ minikube_driver }} --cpus={{ minikube_cpus }} --memory={{ minikube_memory }} --disk-size={{ minikube_disk_size }}"
  environment:
    CHANGE_MINIKUBE_NONE_USER: true
  become_user: "{{ minikube_user }}"
  become: yes
```

### tasks/oraclelinux.yml

```yaml
---
- name: Add Docker CE repository for Oracle Linux
  yum_repository:
    name: docker-ce
    description: Docker CE Repository
    baseurl: https://download.docker.com/linux/centos/$releasever/$basearch/stable
    gpgkey: https://download.docker.com/linux/centos/gpg
    gpgcheck: yes
    enabled: yes

- name: Install Docker CE for Oracle Linux
  package:
    name: docker-ce
    state: present
```

### tasks/ubuntu.yml

```yaml
---
- name: Add Docker GPG key for Ubuntu
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker repository for Ubuntu
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present
    update_cache: yes

- name: Install Docker CE for Ubuntu
  package:
    name: docker-ce
    state: present
```

### vars/oraclelinux.yml

```yaml
---
minikube_dependencies:
  - conntrack
  - socat
  - ebtables
  - ethtool
  - wget
  - curl
  - tar
  - gcc
  - make
  - perl
  - libvirt
  - libvirt-devel
```

### vars/ubuntu.yml

```yaml
---
minikube_dependencies:
  - conntrack
  - socat
  - ebtables
  - ethtool
  - wget
  - curl
  - tar
  - gcc
  - make
  - perl
  - libvirt-clients
  - libvirt-daemon-system
  - qemu-kvm
  - bridge-utils
  - virt-manager
```

### templates/minikube.service.j2

```systemd
[Unit]
Description=Minikube
After=network.target docker.service

[Service]
User={{ minikube_user }}
Group={{ minikube_group }}
ExecStart=/usr/local/bin/minikube start --driver={{ minikube_driver }} --cpus={{ minikube_cpus }} --memory={{ minikube_memory }} --disk-size={{ minikube_disk_size }}
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

## Пример playbook

```yaml
---
- name: Install and configure Minikube
  hosts: all
  become: yes
  roles:
    - minikube
```

## Использование

1. Создайте структуру роли как показано выше
2. Создайте playbook (например, `minikube.yml`)
3. Запустите playbook:

```bash
ansible-playbook -i inventory minikube.yml
```

## Примечания

1. Роль автоматически определяет ОС и применяет соответствующие задачи и переменные.
2. Для работы с KVM на Oracle Linux может потребоваться дополнительная настройка виртуализации.
3. На Ubuntu может потребоваться установка пакета `software-properties-common` перед добавлением репозиториев.
4. Для работы с драйвером docker требуется, чтобы пользователь был в группе docker.

Роль поддерживает установку как последних версий (latest), так и конкретных версий Minikube и kubectl.



Ошибка возникает из-за того, что пакет `libvirt-devel` недоступен в стандартных репозиториях Oracle Linux 9. Давайте исправим роль для корректной работы на Oracle Linux 9.

## Исправления для Oracle Linux 9

### 1. Обновим файл `vars/oraclelinux.yml`:

```yaml
---
minikube_dependencies:
  - conntrack
  - socat
  - ebtables
  - ethtool
  - wget
  - curl
  - tar
  - gcc
  - make
  - perl
  - libvirt
  - virt-install
  - qemu-kvm
  - bridge-utils
```

### 2. Модифицируем `tasks/oraclelinux.yml`:

```yaml
---
- name: Enable KVM module
  modprobe:
    name: kvm
    state: present

- name: Add Docker CE repository for Oracle Linux
  yum_repository:
    name: docker-ce
    description: Docker CE Repository
    baseurl: https://download.docker.com/linux/centos/$releasever/$basearch/stable
    gpgkey: https://download.docker.com/linux/centos/gpg
    gpgcheck: yes
    enabled: yes

- name: Install Docker CE for Oracle Linux
  package:
    name: 
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present

- name: Enable libvirt
  service:
    name: libvirtd
    state: started
    enabled: yes
```

### 3. Добавим проверку перед установкой зависимостей в `tasks/main.yml`:

```yaml
- name: Check if running on Oracle Linux
  stat:
    path: /etc/oracle-release
  register: oracle_check

- name: Install dependencies
  package:
    name: "{{ minikube_dependencies }}"
    state: present
  when: >
    (not oracle_check.stat.exists) or
    (oracle_check.stat.exists and 'libvirt-devel' not in minikube_dependencies)
```

## Альтернативное решение

Если вам всё же нужен пакет `libvirt-devel`, можно добавить EPEL репозиторий:

```yaml
- name: Add EPEL repository for Oracle Linux
  yum_repository:
    name: epel
    description: EPEL Repository
    baseurl: https://download.fedoraproject.org/pub/epel/$releasever/Everything/$basearch
    gpgkey: https://download.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-$releasever
    gpgcheck: yes
    enabled: yes
```

## Рекомендации

1. Для Oracle Linux 9 лучше использовать драйвер `docker` или `podman` вместо KVM, так как настройка KVM может быть сложной.
2. Проверьте, включена ли виртуализация в BIOS:
   ```bash
   grep -E --color '(vmx|svm)' /proc/cpuinfo
   ```
3. Для работы с KVM пользователь должен быть в группе `libvirt`:
   ```yaml
   - name: Add user to libvirt group
     user:
       name: "{{ minikube_user }}"
       groups: libvirt
       append: yes
   ```

После внесения этих изменений роль должна работать корректно на Oracle Linux 9.