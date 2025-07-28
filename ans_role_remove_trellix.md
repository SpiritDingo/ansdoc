# Ansible роль для удаления Trellix (FireEye) с Linux систем

## Структура роли

```
roles/
  remove_trellix_linux/
    tasks/
      main.yml
    handlers/
      main.yml
    vars/
      main.yml
```

## Файлы роли

### vars/main.yml
```yaml
trellix_packages:
  - fireeye
  - xagt
  - fe_av
  - fe_kernel
  - feam
  - fetp

trellix_services:
  - xagt
  - fe_av
  - fe_kernel
  - feam
  - fetp

trellix_dirs:
  - /opt/fireeye
  - /var/opt/fireeye
  - /etc/opt/fireeye
  - /usr/lib/fireeye
  - /usr/lib64/fireeye
```

### tasks/main.yml
```yaml
---
- name: Gather package facts
  package_facts:
    manager: auto

- name: Check if Trellix packages are installed
  debug:
    msg: "Trellix packages detected for removal"
  when: trellix_packages is intersect(ansible_facts.packages)

- name: Stop Trellix services
  systemd:
    name: "{{ item }}"
    state: stopped
    enabled: no
  loop: "{{ trellix_services }}"
  ignore_errors: yes
  when: trellix_packages is intersect(ansible_facts.packages)
  notify:
    - daemon-reload

- name: Remove Trellix packages
  package:
    name: "{{ item }}"
    state: absent
    autoremove: yes
  loop: "{{ trellix_packages }}"
  when: trellix_packages is intersect(ansible_facts.packages)

- name: Remove Trellix directories
  file:
    path: "{{ item }}"
    state: absent
  loop: "{{ trellix_dirs }}"
  ignore_errors: yes

- name: Clean up cron jobs
  cron:
    name: "Trellix related jobs"
    user: root
    state: absent
    job: "* * * * * /opt/fireeye/*"

- name: Remove Trellix kernel modules
  modprobe:
    name: "{{ item }}"
    state: absent
  loop:
    - fe_av
    - fe_kernel
  ignore_errors: yes
```

### handlers/main.yml
```yaml
---
- name: daemon-reload
  systemd:
    daemon_reload: yes
```

## Использование роли

1. Добавьте роль в ваш playbook:

```yaml
- hosts: linux_hosts
  become: yes
  roles:
    - remove_trellix_linux
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini remove_trellix_linux_playbook.yml
```

## Примечания

1. Роль проверяет наличие пакетов Trellix перед выполнением операций удаления
2. Удаляются не только пакеты, но и конфигурационные файлы, каталоги и сервисы
3. Для работы роли требуются права root (указано `become: yes`)
4. В зависимости от версии Trellix/FireEye, имена пакетов и служб могут отличаться
5. Перед использованием в production рекомендуется протестировать роль в тестовой среде
6. Для некоторых дистрибутивов Linux может потребоваться адаптация списка пакетов

Для более полного удаления вы можете добавить дополнительные задачи для очистки:
- временных файлов
- логов
- пользовательских конфигураций
- системных юнитов systemd