### **Ansible-роль для установки и настройки Uyuni Server**  

Эта роль автоматизирует развёртывание **Uyuni Server** (альтернатива Spacewalk с поддержкой `rpm` и `deb`) на **RHEL/CentOS** или **openSUSE**.  

---

## **1. Структура роли**  
```
uyuni-server-role/  
├── defaults/  
│   └── main.yml          # Основные переменные (версия, порты и т. д.)  
├── tasks/  
│   ├── main.yml          # Основные задачи  
│   ├── repo_setup.yml    # Настройка репозиториев Uyuni  
│   └── post_install.yml  # Пост-установочная настройка  
├── templates/  
│   └── uyuni_answer.j2   # Шаблон ответов для автоматической установки  
└── vars/  
    └── main.yml          # Константы (URL репозиториев)  
```

---

## **2. Основные файлы роли**  

### **`defaults/main.yml`**  
```yaml
uyuni_version: "stable"  # или "latest"
uyuni_repo_url: "https://download.opensuse.org/repositories/systemsmanagement:/Uyuni:/{{ uyuni_version }}/RHEL_8/"
uyuni_admin_email: "admin@example.com"
uyuni_admin_pass: "SecurePassword123"
```

### **`tasks/main.yml`**  
```yaml
---
- name: "Установка зависимостей"
  package:
    name:
      - wget
      - curl
      - postgresql
    state: present

- name: "Добавление репозитория Uyuni"
  include_tasks: repo_setup.yml

- name: "Установка Uyuni Server"
  package:
    name: uyuni-server
    state: present

- name: "Запуск автоматической настройки"
  template:
    src: "uyuni_answer.j2"
    dest: "/root/uyuni_answers.txt"
  notify: "Run Uyuni setup"

- meta: flush_handlers
```

### **`tasks/repo_setup.yml`**  
```yaml
- name: "Импорт GPG-ключа Uyuni"
  rpm_key:
    key: "https://download.opensuse.org/repositories/systemsmanagement:/Uyuni:/{{ uyuni_version }}/RHEL_8/repodata/repomd.xml.key"
    state: present

- name: "Добавление репозитория"
  yum_repository:
    name: "uyuni-server"
    description: "Uyuni Server {{ uyuni_version }}"
    baseurl: "{{ uyuni_repo_url }}"
    gpgcheck: yes
    enabled: yes
```

### **`templates/uyuni_answer.j2`**  
```ini
[config]
email = {{ uyuni_admin_email }}
password = {{ uyuni_admin_pass }}
```

### **`handlers/main.yml`**  
```yaml
- name: "Run Uyuni setup"
  command: "/usr/lib/susemanager/bin/mgr-setup -A /root/uyuni_answers.txt"
  become: yes
```

---

## **3. Playbook для применения роли**  
Создайте `playbook.yml`:  
```yaml
- hosts: uyuni_server
  become: yes
  roles:
    - uyuni-server-role
```

---

## **4. Запуск**  
```bash
ansible-playbook -i inventory.ini playbook.yml
```

---

## **5. Проверка установки**  
После установки:  
- Откройте веб-интерфейс: `https://<server-ip>`  
- Логин: `admin`, пароль: `{{ uyuni_admin_pass }}`  

---

## **Дополнительные настройки (опционально)**  
### **Добавление клиентов (SaltStack)**  
```yaml
- name: "Регистрация клиентов через Salt"
  command: "salt-key -A -y"
  when: inventory_hostname == "uyuni-server"
```

### **Синхронизация репозиториев**  
```yaml
- name: "Синхронизация CentOS репозитория"
  command: "mgr-sync refresh"
```

---

## **Итог**  
Эта роль:  
✅ Устанавливает Uyuni Server автоматически.  
✅ Настраивает админ-панель с заданным паролем.  
✅ Подходит для управления `rpm` и `deb` пакетами.  

Для работы требуется:  
- **RHEL/CentOS 8+** или **openSUSE**.  
- **2+ GB RAM, 50+ GB HDD**.  
- Доступ в интернет для загрузки пакетов.  

Если нужно управлять **Debian/Ubuntu**, после установки добавьте `deb`-репозитории через веб-интерфейс Uyuni.