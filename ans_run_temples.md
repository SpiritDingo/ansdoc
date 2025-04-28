### **Примеры запуска Ansible Playbook на определённую группу серверов**

Если в вашем инвентаре есть несколько групп серверов (например, `web`, `db`, `prod`, `dev`), можно запускать playbook **только на выбранную группу**. Вот несколько способов.

---

## **1. Запуск на группу из инвентарного файла**
Допустим, у нас есть `inventory.ini`:
```ini
[web]
web1.example.com
web2.example.com

[db]
db1.example.com
db2.example.com

[prod:children]
web
db

[dev]
dev1.example.com
```

### **Вариант 1: Указание группы в команде (`-l` или `--limit`)**
```bash
ansible-playbook -i inventory.ini playbook.yml --limit "web"
```
→ Запустится **только на серверах из группы `web`**.

### **Вариант 2: Запуск по нескольким группам**
```bash
ansible-playbook -i inventory.ini playbook.yml --limit "web,db"
```
→ Запустится на серверах из групп `web` и `db`.

### **Вариант 3: Исключение группы (`!`)**
```bash
ansible-playbook -i inventory.ini playbook.yml --limit "all:!dev"
```
→ Запустится на **всех серверах, кроме `dev`**.

---

## **2. Запуск на группу через `hosts:` в playbook**
Если в `playbook.yml` указана конкретная группа:
```yaml
- hosts: web  # <- только группа 'web'
  tasks:
    - name: Ensure Nginx is installed
      apt:
        name: nginx
        state: present
```
→ Playbook **автоматически выполнится только на группе `web`**, даже без `--limit`.

---

## **3. Запуск на вложенные группы (например, `prod:children`)**
Если в инвентаре есть группа `prod`, включающая `web` и `db`:
```bash
ansible-playbook -i inventory.ini playbook.yml --limit "prod"
```
→ Запустится на **всех серверах из `web` и `db`**.

---

## **4. Запуск на группу + дополнительные фильтры**
### **Только определённые серверы внутри группы**
```bash
ansible-playbook -i inventory.ini playbook.yml --limit "web:web1.example.com"
```
→ Запустится **только на `web1.example.com` из группы `web`**.

### **Сочетание с тегами (`--tags`)**
```bash
ansible-playbook -i inventory.ini playbook.yml --limit "web" --tags "nginx"
```
→ Запустит **только задачи с тегом `nginx` на группе `web`**.

---

## **5. Динамический инвентарь (AWS, Cloud и др.)**
Если используется динамический инвентарь (например, AWS EC2), можно фильтровать по тегам:
```bash
ansible-playbook -i aws_ec2.yml playbook.yml --limit "tag_group_web"
```
→ Запустится **только на инстансах с тегом `group=web`**.

---

## **Итог: Основные команды**
| Команда | Описание |
|---------|----------|
| `ansible-playbook -i inventory.ini playbook.yml --limit "web"` | Только группа `web` |
| `ansible-playbook -i inventory.ini playbook.yml --limit "web,db"` | Группы `web` и `db` |
| `ansible-playbook -i inventory.ini playbook.yml --limit "all:!dev"` | Все, кроме `dev` |
| `ansible-playbook -i inventory.ini playbook.yml --limit "prod"` | Вложенная группа (`prod:children`) |
| `ansible-playbook -i inventory.ini playbook.yml --limit "web:web1"` | Конкретный сервер в группе |

**Примеры:**
```bash
# Запуск только на production-серверах
ansible-playbook -i inventory.ini deploy.yml --limit "prod"

# Запуск только на web-серверах, кроме web2
ansible-playbook -i inventory.ini deploy.yml --limit "web:!web2.example.com"

# Запуск с тегами на группе db
ansible-playbook -i inventory.ini db_setup.yml --limit "db" --tags "mysql"
```

Таким образом, можно гибко управлять тем, на каких серверах выполняется playbook. 🚀