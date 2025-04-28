# Роль Ansible для установки и настройки Kubernetes кластера (2 мастера + 3 воркера)

Вот структура роли Ansible для развертывания Kubernetes кластера с высокой доступностью (2 мастера) и 3 рабочими узлами.

## Структура роли

```
kubernetes-cluster/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── common.yml
│   ├── masters.yml
│   ├── workers.yml
│   ├── network.yml
│   └── main.yml
├── templates/
│   ├── kubeadm-config.yaml.j2
│   ├── haproxy.cfg.j2
│   └── keepalived.conf.j2
├── files/
│   └── calico.yaml
└── handlers/
    └── main.yml
```

## Основные файлы роли

### defaults/main.yml

```yaml
# Версии компонентов
kube_version: "1.28.0"
docker_version: "20.10"
calico_version: "3.26"

# Настройки сети
pod_network_cidr: "192.168.0.0/16"
service_cidr: "10.96.0.0/12"

# Настройки HA
haproxy_port: 6443
keepalived_virtual_ip: "192.168.1.100"
keepalived_interface: "eth0"
keepalived_router_id: "51"

# Список мастеров и воркеров (заполняется в inventory)
masters: []
workers: []
```

### tasks/main.yml

```yaml
- name: Include common tasks for all nodes
  include_tasks: common.yml
  tags: common

- name: Configure master nodes
  include_tasks: masters.yml
  when: "'master' in group_names"
  tags: masters

- name: Configure worker nodes
  include_tasks: workers.yml
  when: "'worker' in group_names"
  tags: workers

- name: Configure network plugin
  include_tasks: network.yml
  tags: network
```

### tasks/common.yml

```yaml
- name: Disable swap
  command: swapoff -a
  changed_when: false

- name: Disable swap in fstab
  replace:
    path: /etc/fstab
    regexp: '^([^#].*?\sswap\s+sw\s+.*)$'
    replace: '# \1'

- name: Load kernel modules
  modprobe:
    name: "{{ item }}"
    state: present
  loop:
    - br_netfilter
    - overlay

- name: Configure sysctl parameters
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    state: present
    reload: yes
  loop:
    - { key: "net.bridge.bridge-nf-call-iptables", value: "1" }
    - { key: "net.ipv4.ip_forward", value: "1" }

- name: Install required packages
  apt:
    name: "{{ item }}"
    state: present
    update_cache: yes
  loop:
    - apt-transport-https
    - ca-certificates
    - curl
    - gnupg
    - software-properties-common

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker repository
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present

- name: Install Docker
  apt:
    name: "docker-ce={{ docker_version }}.*"
    state: present

- name: Add Kubernetes GPG key
  apt_key:
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
    state: present

- name: Add Kubernetes repository
  apt_repository:
    repo: "deb https://apt.kubernetes.io/ kubernetes-xenial main"
    state: present

- name: Install kubeadm, kubelet and kubectl
  apt:
    name: "{{ item }}={{ kube_version }}-00"
    state: present
  loop:
    - kubeadm
    - kubelet
    - kubectl

- name: Enable and start kubelet
  systemd:
    name: kubelet
    enabled: yes
    state: started
```

### tasks/masters.yml

```yaml
- name: Install HAProxy and Keepalived on first master
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - haproxy
    - keepalived
  when: inventory_hostname == masters[0]

- name: Configure HAProxy
  template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
  when: inventory_hostname == masters[0]
  notify: Restart HAProxy

- name: Configure Keepalived
  template:
    src: keepalived.conf.j2
    dest: /etc/keepalived/keepalived.conf
  when: inventory_hostname == masters[0]
  notify: Restart Keepalived

- name: Initialize first master node
  command: kubeadm init --config /root/kubeadm-config.yaml
  when: inventory_hostname == masters[0]

- name: Copy kubeconfig to user directory
  command: "{{ item }}"
  loop:
    - mkdir -p $HOME/.kube
    - cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    - chown $(id -u):$(id -g) $HOME/.kube/config
  when: inventory_hostname == masters[0]

- name: Generate join command for masters
  command: kubeadm token create --print-join-command
  register: join_command_master
  when: inventory_hostname == masters[0]

- name: Generate join command for workers
  command: kubeadm token create --print-join-command
  register: join_command_worker
  when: inventory_hostname == masters[0]

- name: Join other masters to cluster
  command: "{{ hostvars[masters[0]].join_command_master.stdout }} --control-plane"
  when: inventory_hostname in masters[1:]
```

### tasks/workers.yml

```yaml
- name: Join worker nodes to cluster
  command: "{{ hostvars[masters[0]].join_command_worker.stdout }}"
  when: inventory_hostname in workers
```

### tasks/network.yml

```yaml
- name: Apply Calico network plugin
  command: kubectl apply -f /root/calico.yaml
  when: inventory_hostname == masters[0]
```

### templates/kubeadm-config.yaml.j2

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: "unix:///var/run/containerd/containerd.sock"
  kubeletExtraArgs:
    cgroup-driver: "systemd"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: {{ kube_version }}
controlPlaneEndpoint: "{{ keepalived_virtual_ip }}:{{ haproxy_port }}"
networking:
  podSubnet: {{ pod_network_cidr }}
  serviceSubnet: {{ service_cidr }}
apiServer:
  certSANs:
  - "{{ keepalived_virtual_ip }}"
  - "127.0.0.1"
  - "localhost"
  extraArgs:
    authorization-mode: "Node,RBAC"
```

### templates/haproxy.cfg.j2

```yaml
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

frontend kubernetes
    bind *:{{ haproxy_port }}
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    balance roundrobin
    option tcp-check
{% for master in masters %}
    server k8s-master-{{ loop.index }} {{ master }}:6443 check
{% endfor %}
```

### templates/keepalived.conf.j2

```yaml
vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    interface {{ keepalived_interface }}
    state MASTER
    virtual_router_id {{ keepalived_router_id }}
    priority 100
    advert_int 1

    virtual_ipaddress {
        {{ keepalived_virtual_ip }}
    }

    track_script {
        chk_haproxy
    }
}
```

### handlers/main.yml

```yaml
- name: Restart HAProxy
  service:
    name: haproxy
    state: restarted

- name: Restart Keepalived
  service:
    name: keepalived
    state: restarted
```

## Пример inventory файла

```ini
[masters]
master1 ansible_host=192.168.1.101
master2 ansible_host=192.168.1.102

[workers]
worker1 ansible_host=192.168.1.201
worker2 ansible_host=192.168.1.202
worker3 ansible_host=192.168.1.203

[kube:children]
masters
workers
```

## Использование роли

1. Создайте playbook (например, `kube-cluster.yml`):

```yaml
- hosts: kube
  become: yes
  roles:
    - kubernetes-cluster
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini kube-cluster.yml
```

Эта роль выполняет:
1. Общую настройку всех узлов (отключение swap, настройка ядра, установка Docker и Kubernetes компонентов)
2. Настройку мастер-узлов с HAProxy и Keepalived для высокой доступности
3. Присоединение рабочих узлов к кластеру
4. Установку сетевого плагина Calico

После выполнения кластер будет готов к работе с двумя мастер-нодами (с балансировкой нагрузки через HAProxy и виртуальным IP через Keepalived) и тремя рабочими нодами.