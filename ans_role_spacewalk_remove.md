# Ansible роль для удаления агента Spacewalk с Oracle Linux 9

Вот роль для удаления клиентских компонентов Spacewalk (rhn-client-tools) с Oracle Linux 9.

## Структура роли

```
roles/remove_spacewalk_agent/
├── tasks/
│   └── main.yml
├── handlers/
│   └── main.yml
└── vars/
    └── main.yml
```

## Содержание файлов

### vars/main.yml
```yaml
spacewalk_agent_packages:
  - rhn-client-tools
  - rhn-check
  - rhn-setup
  - rhnlib
  - osad
  - rhncfg
  - rhncfg-actions
  - rhncfg-client
  - rhncfg-management

spacewalk_agent_files:
  - /etc/sysconfig/rhn/
  - /etc/rhn/
  - /var/lib/rhn/
```

### tasks/main.yml
```yaml
---
- name: Остановка служб агента Spacewalk
  ansible.builtin.systemd:
    name: "{{ item }}"
    state: stopped
    enabled: no
  loop:
    - osad
    - rhnsd

- name: Удаление пакетов агента Spacewalk
  ansible.builtin.yum:
    name: "{{ spacewalk_agent_packages }}"
    state: absent
    autoremove: yes
  notify: Очистка зависимостей

- name: Удаление конфигурационных файлов агента
  ansible.builtin.file:
    path: "{{ item }}"
    state: absent
  loop: "{{ spacewalk_agent_files }}"

- name: Удаление cron-заданий Spacewalk
  ansible.builtin.file:
    path: /etc/cron.daily/rhn_check
    state: absent

- name: Удаление репозиториев Spacewalk
  ansible.builtin.file:
    path: "/etc/yum.repos.d/spacewalk-client.repo"
    state: absent
```

### handlers/main.yml
```yaml
---
- name: Очистка зависимостей
  ansible.builtin.command: dnf autoremove -y
```

## Пример playbook

```yaml
---
- name: Удаление агента Spacewalk с Oracle Linux 9
  hosts: spacewalk_clients
  become: yes
  roles:
    - remove_spacewalk_agent
```

## Дополнительные действия

1. **Проверка удаления**:
   ```bash
   rpm -qa | grep rhn
   rpm -qa | grep spacewalk
   ```

2. **Очистка кэша DNF**:
   ```bash
   dnf clean all
   ```

3. **Удаление временных файлов** (опционально):
   ```bash
   rm -rf /var/cache/rhn /var/log/rhn
   ```

4. **Удаление системного идентификатора** (если существует):
   ```bash
   rm -f /etc/sysconfig/rhn/systemid
   ```

Эта роль полностью удаляет клиентские компоненты Spacewalk с Oracle Linux 9, включая:
- Пакеты rhn-client-tools и зависимости
- Конфигурационные файлы
- Службы osad и rhnsd
- Cron-задания
- Клиентские репозитории