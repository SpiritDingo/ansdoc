Вот Ansible роль для **закрытия портов** на Oracle Linux 9 (с использованием `firewalld`):

---

### **Структура роли** (`farewell-cmd-close`)
```bash
farewell-cmd-close/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
└── meta/
    └── main.yml
```

---

### **1. `defaults/main.yml`** (настройки по умолчанию)
```yaml
# Порты для закрытия (формат: "порт/протокол")
farewell_ports_to_close:
  - "8080/tcp"
  - "8443/tcp"

# Зона firewalld (должна совпадать с той, где открывались порты)
farewell_firewalld_zone: "public"

# Исключить критические порты (например, SSH)
farewell_protected_ports:
  - "22/tcp"
```

---

### **2. `tasks/main.yml`** (основные задачи)
```yaml
---
- name: Проверить, что firewalld работает
  ansible.builtin.service:
    name: firewalld
    state: started
  tags: firewall

- name: Закрыть указанные порты (если они не защищены)
  ansible.posix.firewalld:
    port: "{{ item }}"
    zone: "{{ farewell_firewalld_zone }}"
    permanent: yes
    immediate: yes
    state: disabled
  loop: "{{ farewell_ports_to_close }}"
  when: item not in farewell_protected_ports
  notify: reload firewalld
  tags: close_ports

- name: Убедиться, что защищённые порты не тронуты
  ansible.builtin.debug:
    msg: "Порт {{ item }} защищён и не будет закрыт!"
  loop: "{{ farewell_protected_ports }}"
  when: item in farewell_ports_to_close
  tags: safety
```

---

### **3. `handlers/main.yml`** (обработчики)
```yaml
---
- name: reload firewalld
  ansible.builtin.service:
    name: firewalld
    state: reloaded
```

---

### **4. `meta/main.yml`** (информация о роли)
```yaml
---
dependencies: []
galaxy_info:
  author: "Your Name"
  description: "Role for closing ports in Oracle Linux 9 via firewalld"
  license: "MIT"
  platforms:
    - name: OracleLinux
      versions: ["9"]
```

---

### **Как использовать роль**:
1. **Добавьте в `playbook.yml`**:
   ```yaml
   - hosts: oracle_linux_servers
     become: true
     roles:
       - role: farewell-cmd-close
         vars:
           farewell_ports_to_close:
             - "80/tcp"
             - "443/tcp"  # Закроет эти порты, если они не в farewell_protected_ports
   ```

2. **Запуск**:
   ```bash
   ansible-playbook -i inventory.ini playbook.yml --tags close_ports
   ```

---

### **Проверка**:
```bash
# Посмотреть текущие открытые порты:
sudo firewall-cmd --list-ports --zone=public
```

---

### **Дополнительно**:
1. **SELinux**: Если порты были настроены через SELinux, может потребоваться сброс контекста:
   ```yaml
   - name: Сбросить SELinux контекст для портов
     community.selinux.seport:
       ports: "{{ item }}"
       setype: "default_t"
       state: absent
     loop: "{{ farewell_ports_to_close }}"
     when: ansible_selinux.status == 'enabled'
   ```

2. **Логирование**: Добавьте задачу для записи закрытых портов в лог:
   ```yaml
   - name: Записать изменения в лог
     ansible.builtin.copy:
       content: "Закрыты порты: {{ farewell_ports_to_close | join(',') }}"
       dest: /var/log/farewell-cmd.log
       owner: root
       group: root
       mode: 0644
   ```

---

### **Важные предупреждения**:
1. Всегда оставляйте **SSH-порт (22/tcp)** открытым!
2. Если закрываете порты для сервисов (например, HTTP/80), убедитесь, что это не нарушит работу системы.
3. Для максимальной безопасности используйте **точное совпадение зоны** (`farewell_firewalld_zone`), в которой порты были открыты.