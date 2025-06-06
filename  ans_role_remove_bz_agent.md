Добавим удаление конфигурационных файлов и папок агента. Вот обновлённая **Ansible-роль**:

---

### **Обновлённая структура роли**
```bash
roles/
└── remove_agent/
    ├── tasks/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    └── templates/
        └── cleanup_agent.sh.j2  # (опционально, для сложного удаления)
```

---

### **1. Обновлённый `vars/main.yml`**
```yaml
agent_package: "bzsenagent"
agent_config_paths:
  - "/etc/{{ agent_package }}/"
  - "/var/lib/{{ agent_package }}/"
  - "/var/log/{{ agent_package }}/"
  - "/opt/{{ agent_package }}/"
  - "~/.config/{{ agent_package }}/"
```

---

### **2. Обновлённый `tasks/main.yml`**
```yaml
---
- name: "Stop agent service (if exists)"
  ansible.builtin.service:
    name: "{{ agent_package }}"
    state: stopped
    enabled: false
  ignore_errors: true

- name: "Remove agent (YUM-based systems)"
  ansible.builtin.package:
    name: "{{ agent_package }}"
    state: absent
  when: ansible_facts['pkg_manager'] == 'yum'

- name: "Remove agent (APT-based systems)"
  ansible.builtin.package:
    name: "{{ agent_package }}"
    state: absent
  when: ansible_facts['pkg_manager'] == 'apt'

- name: "Kill remaining agent processes"
  ansible.builtin.shell: "pkill -9 -f '{{ agent_package }}' || true"
  ignore_errors: true

- name: "Remove agent configuration files and directories"
  ansible.builtin.file:
    path: "{{ item }}"
    state: absent
  loop: "{{ agent_config_paths }}"
  ignore_errors: true

- name: "Remove systemd service (if exists)"
  ansible.builtin.file:
    path: "/etc/systemd/system/{{ agent_package }}.service"
    state: absent
  ignore_errors: true

- name: "Reload systemd (if service was removed)"
  ansible.builtin.systemd:
    daemon_reload: yes
  when: "'/etc/systemd/system/' in agent_config_paths"
```

---

### **3. Дополнительные улучшения**

#### **Шаблон для сложного удаления (`templates/cleanup_agent.sh.j2`)**
```bash
#!/bin/bash
# Полная очистка агента {{ agent_package }}

# Удаление пользователя (если создавался)
userdel -r "{{ agent_package }}" 2>/dev/null || true

# Очистка cron-заданий
crontab -u root -l | grep -v "{{ agent_package }}" | crontab -u root -
```

Добавьте в `tasks/main.yml`:
```yaml
- name: "Deploy and execute cleanup script"
  ansible.builtin.template:
    src: "cleanup_agent.sh.j2"
    dest: "/tmp/cleanup_agent.sh"
    mode: 0744
- ansible.builtin.command: "/tmp/cleanup_agent.sh"
  args:
    removes: "/tmp/cleanup_agent.sh"
```

---

### **4. Пример Playbook**
```yaml
---
- hosts: all
  become: true
  roles:
    - remove_agent
```

---

### **Что делает эта роль:**
1. Останавливает и отключает сервис агента.
2. Удаляет пакет через `yum` или `apt`.
3. Убивает оставшиеся процессы.
4. Удаляет конфиги, логи и данные из указанных путей.
5. Чистит systemd-юнит (если есть).
6. Опционально — удаляет пользователя и cron-задачи.

> **Важно:** Перед применением проверьте пути в `agent_config_paths` для вашего агента!