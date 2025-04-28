# Роль Ansible для установки и настройки кластера Kubernetes (2 мастера, 3 воркера)

## Структура роли

```
kubernetes-cluster/
├── defaults/
│   └── main.yml       # Переменные по умолчанию
├── files/
│   ├── kubeadm-config.yaml  # Конфиг kubeadm
│   └── calico.yaml    # Манифест Calico CNI
├── tasks/
│   ├── common.yml     # Общие задачи для всех узлов
│   ├── master.yml     # Задачи для мастер-узлов
│   ├── worker.yml     # Задачи для воркер-узлов
│   ├── ha.yml         # Настройка HA для мастеров
│   └── main.yml       # Основной файл задач
├── templates/
│   └── keepalived.conf.j2  # Шаблон keepalived
└── vars/
    └── main.yml       # Основные переменные
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Версия Kubernetes
kube_version: "1.28.0"

# Настройки сети
pod_network_cidr: "192.168.0.0/16"
service_cidr: "10.96.0.0/12"

# Настройки HA
virtual_ip: "192.168.1.100"
virtual_interface: "eth0"
keepalived_priority_master1: 100
keepalived_priority_master2: 90
keepalived_router_id: 51
```

### vars/main.yml

```yaml
---
# Список мастер-узлов
master_nodes:
  - name: master1
    ip: 192.168.1.101
    keepalived_priority: "{{ keepalived_priority_master1 }}"
  - name: master2
    ip: 192.168.1.102
    keepalived_priority: "{{ keepalived_priority_master2 }}"

# Список воркер-узлов
worker_nodes:
  - name: worker1
    ip: 192.168.1.103
  - name: worker2
    ip: 192.168.1.104
  - name: worker3
    ip: 192.168.1.105
```

### tasks/main.yml

```yaml
---
- name: Include common tasks for all nodes
  include_tasks: common.yml
  tags: common

- name: Include tasks for master nodes
  include_tasks: master.yml
  when: "'master' in group_names"
  tags: master

- name: Include HA setup for master nodes
  include_tasks: ha.yml
  when: "'master' in group_names"
  tags: ha

- name: Include tasks for worker nodes
  include_tasks: worker.yml
  when: "'worker' in group_names"
  tags: worker
```

### tasks/common.yml

```yaml
---
- name: Disable swap
  shell: |
    swapoff -a
    sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
  become: yes

- name: Load kernel modules
  modprobe:
    name: "{{ item }}"
    state: present
  with_items:
    - br_netfilter
    - overlay
  become: yes

- name: Configure sysctl
  sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    state: present
    reload: yes
  with_items:
    - { name: net.bridge.bridge-nf-call-iptables, value: 1 }
    - { name: net.ipv4.ip_forward, value: 1 }
  become: yes

- name: Install dependencies
  apt:
    name: "{{ item }}"
    state: present
    update_cache: yes
  with_items:
    - apt-transport-https
    - ca-certificates
    - curl
    - gnupg
    - software-properties-common
    - keepalived
    - haproxy
  become: yes

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present
  become: yes

- name: Add Docker repository
  apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable
    state: present
  become: yes

- name: Install Docker
  apt:
    name: docker-ce=5:20.10.23~3-0~ubuntu-{{ ansible_distribution_release }}
    state: present
  become: yes

- name: Add Kubernetes GPG key
  apt_key:
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
    state: present
  become: yes

- name: Add Kubernetes repository
  apt_repository:
    repo: deb https://apt.kubernetes.io/ kubernetes-xenial main
    state: present
    filename: kubernetes
  become: yes

- name: Install kubeadm, kubelet and kubectl
  apt:
    name: "{{ item }}={{ kube_version }}-00"
    state: present
  with_items:
    - kubeadm
    - kubelet
    - kubectl
  become: yes

- name: Enable and start kubelet
  systemd:
    name: kubelet
    enabled: yes
    state: started
  become: yes
```

### tasks/master.yml

```yaml
---
- name: Create kubeadm config directory
  file:
    path: /etc/kubeadm
    state: directory
  become: yes

- name: Copy kubeadm config
  copy:
    src: kubeadm-config.yaml
    dest: /etc/kubeadm/kubeadm-config.yaml
  become: yes

- name: Initialize first master
  command: kubeadm init --config=/etc/kubeadm/kubeadm-config.yaml --upload-certs
  when: inventory_hostname == master_nodes[0].name
  become: yes
  register: kubeadm_init

- name: Save join command for masters
  set_fact:
    master_join_command: "{{ kubeadm_init.stdout | regex_search('kubeadm join .* --control-plane') }}"
  when: inventory_hostname == master_nodes[0].name

- name: Save join command for workers
  set_fact:
    worker_join_command: "{{ kubeadm_init.stdout | regex_search('kubeadm join .* --token') }}"
  when: inventory_hostname == master_nodes[0].name

- name: Join other masters to cluster
  command: "{{ master_join_command }}"
  when: inventory_hostname != master_nodes[0].name and master_join_command is defined
  become: yes

- name: Configure kubectl for root
  block:
    - name: Create .kube directory
      file:
        path: /root/.kube
        state: directory
      become: yes

    - name: Copy admin.conf
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /root/.kube/config
        remote_src: yes
      become: yes
  when: "'master' in group_names"

- name: Install Calico CNI
  command: kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
  when: inventory_hostname == master_nodes[0].name
  become: yes
```

### tasks/ha.yml

```yaml
---
- name: Configure keepalived
  template:
    src: keepalived.conf.j2
    dest: /etc/keepalived/keepalived.conf
  become: yes

- name: Enable and start keepalived
  systemd:
    name: keepalived
    enabled: yes
    state: started
  become: yes

- name: Configure haproxy
  copy:
    content: |
      global
          log /dev/log    local0
          log /dev/log    local1 notice
          chroot /var/lib/haproxy
          stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
          stats timeout 30s
          user haproxy
          group haproxy
          daemon

      defaults
          log     global
          mode    tcp
          option  tcplog
          option  dontlognull
          timeout connect 5000
          timeout client  50000
          timeout server  50000

      frontend k8s-api
          bind *:6443
          default_backend k8s-api

      backend k8s-api
          option tcp-check
          balance roundrobin
          {% for master in master_nodes %}
          server {{ master.name }} {{ master.ip }}:6443 check
          {% endfor %}
    dest: /etc/haproxy/haproxy.cfg
  become: yes

- name: Enable and start haproxy
  systemd:
    name: haproxy
    enabled: yes
    state: started
  become: yes
```

### tasks/worker.yml

```yaml
---
- name: Join worker to cluster
  command: "{{ hostvars[master_nodes[0].name].worker_join_command }}"
  when: worker_join_command is defined
  become: yes
```

### templates/keepalived.conf.j2

```jinja2
vrrp_script chk_haproxy {
    script "pidof haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state {% if inventory_hostname == master_nodes[0].name %}MASTER{% else %}BACKUP{% endif %}
    interface {{ virtual_interface }}
    virtual_router_id {{ keepalived_router_id }}
    priority {{ hostvars[inventory_hostname].keepalived_priority | default(80) }}
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 42
    }
    virtual_ipaddress {
        {{ virtual_ip }}
    }
    track_script {
        chk_haproxy
    }
}
```

### files/kubeadm-config.yaml

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: "unix:///var/run/dockershim.sock"
  kubeletExtraArgs:
    cloud-provider: "external"

---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: {{ kube_version }}
controlPlaneEndpoint: "{{ virtual_ip }}:6443"
apiServer:
  certSANs:
    - "{{ virtual_ip }}"
    {% for master in master_nodes %}
    - "{{ master.ip }}"
    - "{{ master.name }}"
    {% endfor %}
networking:
  podSubnet: "{{ pod_network_cidr }}"
  serviceSubnet: "{{ service_cidr }}"
```

## Инвентарный файл (inventory.ini)

```ini
[master]
master1 ansible_host=192.168.1.101
master2 ansible_host=192.168.1.102

[worker]
worker1 ansible_host=192.168.1.103
worker2 ansible_host=192.168.1.104
worker3 ansible_host=192.168.1.105

[kube_cluster:children]
master
worker
```

## Использование роли

1. Создайте playbook (например, `setup-k8s