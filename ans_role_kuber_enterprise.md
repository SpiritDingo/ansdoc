# Установка и настройка AWX с плагинами в Minikube (корпоративное решение)

## Подготовка окружения

1. Установите Minikube и необходимые зависимости:

```bash
# Установка kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Установка Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Запуск Minikube (пример для корпоративного прокси)
minikube start --driver=docker --cpus=4 --memory=8g --disk-size=50g \
  --docker-env http_proxy=http://corporate.proxy:port \
  --docker-env https_proxy=http://corporate.proxy:port \
  --docker-env no_proxy=localhost,127.0.0.1,.corp.domain
```

## Ansible роль для установки AWX

Создайте структуру роли:
```
roles/awx-minikube/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── awx-values.yml.j2
└── vars
    └── main.yml
```

### tasks/main.yml
```yaml
---
- name: Установка зависимостей
  become: yes
  package:
    name:
      - git
      - docker
      - python3-pip
    state: present

- name: Установка helm
  get_url:
    url: https://get.helm.sh/helm-v3.12.0-linux-amd64.tar.gz
    dest: /tmp/helm.tar.gz
    mode: '0755'
  register: helm_download
  until: helm_download is succeeded
  retries: 3
  delay: 10

- name: Распаковка helm
  unarchive:
    src: /tmp/helm.tar.gz
    dest: /tmp/
    remote_src: yes
    extra_opts: [--strip-components=1]

- name: Копирование helm в /usr/local/bin
  copy:
    src: "/tmp/linux-amd64/helm"
    dest: "/usr/local/bin/helm"
    mode: '0755'
    remote_src: yes

- name: Добавление репозитория AWX
  command: helm repo add awx-operator https://ansible.github.io/awx-operator/
  args:
    creates: "{{ helm_config_path }}/repositories.lock"

- name: Обновление репозиториев helm
  command: helm repo update

- name: Создание namespace для AWX
  command: kubectl create namespace awx
  ignore_errors: yes

- name: Установка AWX Operator
  command: helm install awx-operator awx-operator/awx-operator -n awx
  when: not awx_operator_installed.stat.exists

- name: Применение конфигурации AWX
  template:
    src: awx-values.yml.j2
    dest: /tmp/awx-values.yml
  notify: Применить конфигурацию AWX

- name: Ожидание готовности AWX
  command: kubectl get pods -n awx
  register: awx_status
  until: "'awx-postgres' in awx_status.stdout and 'awx-web' in awx_status.stdout and 'awx-task' in awx_status.stdout"
  retries: 20
  delay: 30

- name: Получение пароля admin
  command: kubectl get secret awx-admin-password -n awx -o jsonpath='{.data.password}' | base64 --decode
  register: awx_admin_password
  changed_when: false
  no_log: true

- name: Сохранение пароля admin
  copy:
    content: "{{ awx_admin_password.stdout }}"
    dest: /root/awx-admin-password.txt
    mode: '0600'
```

### templates/awx-values.yml.j2
```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: NodePort
  ingress_type: none
  hostname: awx.minikube.local
  postgres_configuration_secret: awx-postgres-configuration
  postgres_storage_requirements:
    requests:
      storage: {{ awx_postgres_storage_size }}
  projects_persistence: true
  projects_storage_requirements:
    requests:
      storage: {{ awx_projects_storage_size }}
  extra_volumes: |
    - name: corporate-ca
      configMap:
        name: corporate-ca
  web_extra_volume_mounts: |
    - name: corporate-ca
      mountPath: /etc/pki/ca-trust/source/anchors/corporate-ca.crt
      subPath: corporate-ca.crt
  task_extra_volume_mounts: |
    - name: corporate-ca
      mountPath: /etc/pki/ca-trust/source/anchors/corporate-ca.crt
      subPath: corporate-ca.crt
  ee_extra_volumes: |
    - name: corporate-ca
      configMap:
        name: corporate-ca
  ee_extra_volume_mounts: |
    - name: corporate-ca
      mountPath: /etc/pki/ca-trust/source/anchors/corporate-ca.crt
      subPath: corporate-ca.crt
  control_plane_ee_image: quay.io/ansible/awx-ee:latest
  task_args: >
    ['--inventory', '/inventory/tower_inventory.json']
  web_args: >
    ['--inventory', '/inventory/tower_inventory.json']
```

### defaults/main.yml
```yaml
---
awx_postgres_storage_size: 8Gi
awx_projects_storage_size: 20Gi
awx_admin_user: admin
awx_node_port: 30080
```

## Установка плагинов и корпоративная настройка

Добавьте следующие задачи в роль:

```yaml
- name: Создание ConfigMap с корпоративным CA
  command: |
    kubectl create configmap corporate-ca -n awx \
      --from-file=corporate-ca.crt=/etc/pki/ca-trust/source/anchors/corporate-ca.crt

- name: Установка корпоративных proxy настроек
  command: |
    kubectl set env -n awx deploy/awx-task \
      http_proxy=http://corporate.proxy:port \
      https_proxy=http://corporate.proxy:port \
      no_proxy=localhost,127.0.0.1,.corp.domain

- name: Установка плагинов (пример для VMware)
  command: |
    kubectl exec -n awx deploy/awx-task -- ansible-galaxy collection install \
      vmware.vmware_rest \
      cloud.common \
      --pipelinx-url https://galaxy.ansible.com/api/

- name: Настройка корпоративного репозитория Execution Environments
  command: |
    kubectl exec -n awx deploy/awx-task -- ansible-config set \
      DEFAULT_EE_IMAGE_REGISTRY=corporate.registry:5000 \
      DEFAULT_EE_IMAGE_NAME=ansible/awx-ee-custom
```

## Развертывание

1. Создайте playbook `deploy-awx.yml`:

```yaml
---
- hosts: localhost
  become: yes
  roles:
    - awx-minikube
```

2. Запустите playbook:

```bash
ansible-playbook deploy-awx.yml
```

## Доступ к AWX

После установки:
1. Получите IP Minikube: `minikube ip`
2. Получите NodePort AWX: `kubectl get svc -n awx awx-web`
3. Доступ через: `http://<minikube-ip>:<nodeport>`
4. Логин: admin
5. Пароль: содержимое `/root/awx-admin-password.txt`

## Дополнительные корпоративные настройки

Для enterprise-среды рекомендуется:
1. Настроить LDAP/AD интеграцию
2. Включить TLS
3. Настроить резервное копирование
4. Реализовать мониторинг
5. Настроить интеграцию с корпоративным VCS (GitLab, GitHub Enterprise и т.д.)

Все эти настройки могут быть добавлены в роль как дополнительные задачи и шаблоны конфигураций.





Вот готовая Ansible роль для установки Minikube и зависимостей:

## Структура роли
```
roles/install-minikube/
├── tasks
│   └── main.yml
├── defaults
│   └── main.yml
└── vars
    └── main.yml
```

### tasks/main.yml
```yaml
---
- name: Установка зависимостей
  become: yes
  package:
    name:
      - curl
      - conntrack
      - socat
      - ebtables
      - ethtool
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present
  when: ansible_os_family == "Debian"

- name: Добавление Docker репозитория (для Debian/Ubuntu)
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/{{ ansible_distribution|lower }} {{ ansible_distribution_release }} stable"
    state: present
    filename: docker-ce
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Установка зависимостей для RHEL
  package:
    name:
      - curl
      - conntrack
      - socat
      - ebtables
      - ethtool
      - docker
    state: present
  when: ansible_os_family == "RedHat"

- name: Запуск и включение Docker
  service:
    name: docker
    state: started
    enabled: yes

- name: Скачивание kubectl
  get_url:
    url: "https://dl.k8s.io/release/{{ kubectl_version }}/bin/linux/amd64/kubectl"
    dest: "/tmp/kubectl"
    mode: '0755'
    validate_certs: no
  register: download_kubectl
  until: download_kubectl is succeeded
  retries: 3
  delay: 10

- name: Установка kubectl
  copy:
    src: "/tmp/kubectl"
    dest: "/usr/local/bin/kubectl"
    remote_src: yes
    mode: '0755'

- name: Проверка версии kubectl
  command: kubectl version --client --short
  register: kubectl_version_check
  changed_when: false

- name: Скачивание Minikube
  get_url:
    url: "https://storage.googleapis.com/minikube/releases/{{ minikube_version }}/minikube-linux-amd64"
    dest: "/tmp/minikube"
    mode: '0755'
    validate_certs: no
  register: download_minikube
  until: download_minikube is succeeded
  retries: 3
  delay: 10

- name: Установка Minikube
  copy:
    src: "/tmp/minikube"
    dest: "/usr/local/bin/minikube"
    remote_src: yes
    mode: '0755'

- name: Проверка версии Minikube
  command: minikube version
  register: minikube_version_check
  changed_when: false

- name: Запуск Minikube
  command: "minikube start --driver={{ minikube_driver }} --cpus={{ minikube_cpus }} --memory={{ minikube_memory }}"
  environment:
    HTTP_PROXY: "{{ http_proxy | default('') }}"
    HTTPS_PROXY: "{{ https_proxy | default('') }}"
    NO_PROXY: "{{ no_proxy | default('') }}"
  register: minikube_start
  retries: 3
  delay: 60
  until: minikube_start is succeeded

- name: Проверка статуса Minikube
  command: minikube status
  register: minikube_status
  changed_when: false
```

### defaults/main.yml
```yaml
---
minikube_version: "latest"
kubectl_version: "stable"
minikube_driver: "docker"
minikube_cpus: 2
minikube_memory: "4g"
```

### vars/main.yml
```yaml
---
# Прокси-настройки для корпоративной среды
http_proxy: ""
https_proxy: ""
no_proxy: ""
```

## Пример playbook для использования роли

Создайте файл `install-minikube.yml`:
```yaml
---
- hosts: target_servers
  become: yes
  vars:
    http_proxy: "http://corporate.proxy:3128"
    https_proxy: "http://corporate.proxy:3128"
    no_proxy: "localhost,127.0.0.1,.corp.domain"
  roles:
    - install-minikube
```

## Использование
1. Установите роль в ваш проект:
```bash
mkdir -p roles
git clone https://github.com/your-repo/install-minikube.git roles/install-minikube
```

2. Запустите playbook:
```bash
ansible-playbook -i inventory.ini install-minikube.yml
```

## Особенности роли:
- Поддержка Debian/Ubuntu и RHEL/CentOS
- Настройка прокси для корпоративных сред
- Возможность указания конкретных версий kubectl и Minikube
- Проверка установленных компонентов
- Автоматический запуск Minikube после установки
- Возможность настройки параметров Minikube (CPU, память, драйвер)

Для корпоративного использования рекомендуется:
1. Заменить URL загрузки на внутренние зеркала
2. Добавить корпоративные сертификаты
3. Настроить внутренние DNS имена в no_proxy