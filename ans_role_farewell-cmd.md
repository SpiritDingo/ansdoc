Вот Ansible роль для настройки **farewell-cmd** с открытием портов на **Oracle Linux 9** (аналогично RHEL 9, использует `firewalld` по умолчанию):

---

### **Структура роли**:
```bash
farewell-cmd/
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
# Порты для открытия (формат: "порт/протокол")
farewell_ports:
  - "80/tcp"
  - "443/tcp"
  - "22/tcp"  # SSH (убедитесь, что не закрыли его!)

# Зона firewalld (public, internal, trusted и т.д.)
farewell_firewalld_zone: "public"

# Включить firewalld (если выключен)
farewell_enable_firewalld: true
```

---

### **2. `tasks/main.yml`** (основные задачи)
```yaml
---
- name: Убедиться, что firewalld установлен
  ansible.builtin.package:
    name: firewalld
    state: present
  tags: firewall

- name: Включить и запустить firewalld
  ansible.builtin.service:
    name: firewalld
    enabled: "{{ farewell_enable_firewalld }}"
    state: started
  tags: firewall

- name: Открыть порты в firewalld
  ansible.posix.firewalld:
    port: "{{ item }}"
    zone: "{{ farewell_firewalld_zone }}"
    permanent: yes
    immediate: yes  # применить без перезагрузки
    state: enabled
  loop: "{{ farewell_ports }}"
  notify: reload firewalld
  tags: ports
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
  description: "Role for opening ports in Oracle Linux 9 via firewalld"
  license: "MIT"
  platforms:
    - name: OracleLinux
      versions: ["9"]
```

---

### **Использование роли**:
1. **Добавьте роль в `playbook.yml`**:
   ```yaml
   - hosts: oracle_linux_servers
     become: true  # Требуются права root
     roles:
       - role: farewell-cmd
         vars:
           farewell_ports:
             - "8080/tcp"
             - "8443/tcp"
   ```

2. **Запустите playbook**:
   ```bash
   ansible-playbook -i inventory.ini playbook.yml
   ```

---

### **Проверка**:
После выполнения роли проверьте открытые порты:
```bash
sudo firewall-cmd --list-ports --zone=public
# Или полный список правил:
sudo firewall-cmd --list-all
```

---

### **Важно**:
1. Oracle Linux 9 использует `firewalld` по умолчанию (аналогично RHEL 9).
2. Если нужно открыть порты для **UDP**, укажите это явно (например, `"53/udp"`).
3. Убедитесь, что не закрываете SSH-порт (`22/tcp`), иначе потеряете доступ к серверу!

Для более сложных сценариев (например, настройки SELinux) можно расширить роль дополнительными задачами.