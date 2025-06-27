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




Ошибка указывает, что Minikube не был установлен или не находится в PATH. Давайте исправим роль, чтобы гарантировать правильную установку Minikube. Вот исправления:

### 1. Обновим tasks/main.yml (исправленная часть установки Minikube):

```yaml
- name: Install latest minikube (alternative method)
  block:
    - name: Download minikube
      get_url:
        url: https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
        dest: /tmp/minikube
        mode: '0755'
      
    - name: Install minikube to /usr/local/bin
      copy:
        src: /tmp/minikube
        dest: /usr/local/bin/minikube
        remote_src: yes
        mode: '0755'
      
    - name: Verify minikube installation
      command: minikube version
      register: minikube_check
      changed_when: false
      
    - name: Debug minikube version
      debug:
        var: minikube_check.stdout
      
    - name: Cleanup temp file
      file:
        path: /tmp/minikube
        state: absent
  when: minikube_version == "latest"
```

### 2. Добавим проверку PATH в начале tasks:

```yaml
- name: Ensure /usr/local/bin is in PATH
  lineinfile:
    path: /etc/environment
    regexp: '^PATH='
    line: 'PATH="/usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/sbin:/sbin"'
    state: present
  when: ansible_distribution != 'Ubuntu'  # Ubuntu обычно уже имеет правильный PATH

- name: Update environment variables
  shell: source /etc/environment
```

### 3. Добавим проверку установки после шага:

```yaml
- name: Verify minikube is executable
  stat:
    path: /usr/local/bin/minikube
  register: minikube_bin

- name: Fail if minikube not installed
  fail:
    msg: "Minikube binary not found at /usr/local/bin/minikube after installation"
  when: not minikube_bin.stat.exists
```

### 4. Для драйвера Docker добавим проверку:

```yaml
- name: Verify Docker is running
  command: docker ps
  register: docker_check
  ignore_errors: yes

- name: Fail if Docker not working
  fail:
    msg: "Docker is not running properly. Please check Docker installation."
  when: docker_check.rc != 0
```

### 5. Альтернативный способ запуска Minikube:

```yaml
- name: Start minikube (with retry)
  command: "minikube start --driver={{ minikube_driver }} --cpus={{ minikube_cpus }} --memory={{ minikube_memory }} --disk-size={{ minikube_disk_size }}"
  environment:
    CHANGE_MINIKUBE_NONE_USER: true
  become_user: "{{ minikube_user }}"
  become: yes
  register: minikube_start
  retries: 3
  delay: 10
  until: minikube_start.rc == 0
  ignore_errors: yes

- name: Debug minikube start output
  debug:
    var: minikube_start
```

### Полное исправление для проблемы:

1. Убедитесь, что /usr/local/bin в PATH:
```bash
echo $PATH
```

2. Проверьте наличие minikube:
```bash
ls -la /usr/local/bin/minikube
```

3. Если проблема сохраняется, попробуйте установить вручную:
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

4. Проверьте права:
```bash
sudo chmod +x /usr/local/bin/minikube
```

После этих изменений роль должна:
1. Корректно устанавливать Minikube
2. Проверять его наличие
3. Правильно запускать с указанным драйвером
4. Иметь механизм повтора при неудачном запуске

Если проблема сохраняется, проверьте:
- Доступ к storage.googleapis.com (может быть заблокирован брандмауэром)
- Достаточно ли прав у пользователя
- Достаточно ли ресурсов на машине для запуска Minikube


Эта ошибка возникает, когда Minikube пытается прочитать конфигурационный файл, который еще не существует. Давайте исправим это в Ansible роли.

## Решение проблемы

### 1. Добавим задачу для создания директории конфигурации

В файл `tasks/main.yml` добавим:

```yaml
- name: Ensure minikube config directory exists
  file:
    path: "~{{ minikube_user }}/.minikube/config"
    state: directory
    owner: "{{ minikube_user }}"
    group: "{{ minikube_group }}"
    mode: '0755'
```

### 2. Модифицируем задачу запуска Minikube

Обновим задачу запуска Minikube:

```yaml
- name: Start minikube
  command: "minikube start --driver={{ minikube_driver }} --cpus={{ minikube_cpus }} --memory={{ minikube_memory }} --disk-size={{ minikube_disk_size }}"
  environment:
    CHANGE_MINIKUBE_NONE_USER: true
    HOME: "~{{ minikube_user }}"
  become_user: "{{ minikube_user }}"
  become: yes
  args:
    creates: "~{{ minikube_user }}/.minikube/config/config.json"
```

### 3. Добавим обработку ошибок

Добавим блок обработки ошибок:

```yaml
- name: Initialize minikube config if needed
  block:
    - name: Create empty config if missing
      copy:
        content: "{}"
        dest: "~{{ minikube_user }}/.minikube/config/config.json"
        owner: "{{ minikube_user }}"
        group: "{{ minikube_group }}"
        mode: '0644'
      when: not minikube_config.stat.exists
      
    - name: Set correct permissions
      file:
        path: "~{{ minikube_user }}/.minikube"
        state: directory
        owner: "{{ minikube_user }}"
        group: "{{ minikube_group }}"
        recurse: yes
  rescue:
    - name: Debug minikube config error
      debug:
        msg: "Failed to initialize minikube config, trying alternative approach"
      
    - name: Run minikube version to initialize config
      command: minikube version
      become_user: "{{ minikube_user }}"
      become: yes
```

### 4. Полный исправленный блок для Minikube

```yaml
- name: Ensure minikube environment is ready
  block:
    - name: Create minikube config directory
      file:
        path: "~{{ minikube_user }}/.minikube/config"
        state: directory
        owner: "{{ minikube_user }}"
        group: "{{ minikube_group }}"
        mode: '0755'
      
    - name: Check if config exists
      stat:
        path: "~{{ minikube_user }}/.minikube/config/config.json"
      register: minikube_config
      
    - name: Initialize empty config if missing
      copy:
        content: "{}"
        dest: "~{{ minikube_user }}/.minikube/config/config.json"
        owner: "{{ minikube_user }}"
        group: "{{ minikube_group }}"
        mode: '0644'
      when: not minikube_config.stat.exists
      
    - name: Run minikube version to initialize
      command: minikube version
      become_user: "{{ minikube_user }}"
      become: yes
      changed_when: false
      
- name: Start minikube cluster
  command: "minikube start --driver={{ minikube_driver }} --cpus={{ minikube_cpus }} --memory={{ minikube_memory }} --disk-size={{ minikube_disk_size }}"
  environment:
    HOME: "~{{ minikube_user }}"
  become_user: "{{ minikube_user }}"
  become: yes
  register: minikube_start
  retries: 3
  delay: 10
  until: minikube_start.rc == 0
```

## Дополнительные проверки

1. Убедитесь, что используется правильный пользователь:
```bash
echo $USER
```

2. Проверьте права на директорию minikube:
```bash
ls -la ~/.minikube/
```

3. Если проблема сохраняется, попробуйте выполнить вручную:
```bash
mkdir -p ~/.minikube/config
echo "{}" > ~/.minikube/config/config.json
chown -R $USER:$USER ~/.minikube
```

Эти изменения гарантируют, что:
1. Директория конфигурации будет создана
2. Конфигурационный файл будет инициализирован
3. Права доступа будут установлены правильно
4. Minikube сможет записывать свою конфигурацию