# Ansible роль для обновления Defender на Ubuntu и Oracle Linux 9

Вот пример Ansible роли для обновления Microsoft Defender для Endpoint (MDE) на системах Ubuntu и Oracle Linux 9.

## Структура роли

```
roles/update_defender/
├── tasks/
│   ├── main.yml
│   ├── ubuntu.yml
│   ├── oracle_linux.yml
├── vars/
│   └── main.yml
└── README.md
```

## Файлы роли

### vars/main.yml

```yaml
# Параметры для Defender
defender_package_name: mdatp
defender_repo_url: "https://packages.microsoft.com/config/{{ ansible_distribution | lower }}/{{ ansible_distribution_version }}/prod.list"
```

### tasks/main.yml

```yaml
- name: Include OS-specific tasks
  include_tasks: "{{ ansible_distribution | lower }}.yml"
  when: ansible_distribution in ['Ubuntu', 'OracleLinux']
  
- name: Fail if unsupported OS
  fail:
    msg: "Unsupported OS: {{ ansible_distribution }}"
  when: ansible_distribution not in ['Ubuntu', 'OracleLinux']
```

### tasks/ubuntu.yml

```yaml
- name: Ensure dependencies are installed (Ubuntu)
  apt:
    name: ["curl", "apt-transport-https"]
    state: present
    update_cache: yes

- name: Add Microsoft GPG key (Ubuntu)
  apt_key:
    url: https://packages.microsoft.com/keys/microsoft.asc
    state: present

- name: Add Microsoft Defender repository (Ubuntu)
  apt_repository:
    repo: "deb [arch=amd64] https://packages.microsoft.com/ubuntu/{{ ansible_distribution_version }}/prod {{ ansible_distribution_release }} main"
    state: present
    filename: microsoft-defender.list

- name: Update package cache (Ubuntu)
  apt:
    update_cache: yes

- name: Install or update Microsoft Defender (Ubuntu)
  apt:
    name: "{{ defender_package_name }}"
    state: latest
    update_cache: yes
```

### tasks/oracle_linux.yml

```yaml
- name: Ensure dependencies are installed (Oracle Linux)
  dnf:
    name: ["curl", "yum-utils"]
    state: present

- name: Add Microsoft GPG key (Oracle Linux)
  rpm_key:
    state: present
    key: https://packages.microsoft.com/keys/microsoft.asc

- name: Add Microsoft Defender repository (Oracle Linux)
  yum_repository:
    name: microsoft-defender
    description: Microsoft Defender for Endpoint
    baseurl: https://packages.microsoft.com/oracle/{{ ansible_distribution_version }}/prod/
    gpgkey: https://packages.microsoft.com/keys/microsoft.asc
    gpgcheck: yes
    enabled: yes

- name: Install or update Microsoft Defender (Oracle Linux)
  dnf:
    name: "{{ defender_package_name }}"
    state: latest
    update_cache: yes
```

## Пример playbook

```yaml
- name: Update Microsoft Defender on all servers
  hosts: all
  become: yes
  roles:
    - update_defender
```

## Использование

1. Создайте структуру каталогов как показано выше
2. Заполните файлы содержимым
3. Создайте playbook или добавьте роль в существующий
4. Запустите playbook:

```bash
ansible-playbook -i inventory.ini update_defender.yml
```

## Примечания

1. Убедитесь, что у вас есть соответствующие лицензии для Microsoft Defender для Endpoint
2. Для работы может потребоваться настройка прокси, если ваши серверы находятся за firewall
3. Для Oracle Linux 9 убедитесь, что вы используете правильный major version (9.x)
4. Перед применением в production протестируйте роль в тестовой среде

Вы можете расширить эту роль, добавив задачи для проверки статуса Defender после обновления или для перезагрузки сервиса при необходимости.