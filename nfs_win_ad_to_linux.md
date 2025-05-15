### **Монтирование NFS из Windows в openSUSE с использованием доменного пользователя (Active Directory)**

Чтобы смонтировать NFS-шару с Windows на openSUSE с корректными правами доменного пользователя, нужно:  
1. **Настроить NFS на Windows** с правильными UID/GID или Kerberos-аутентификацией.  
2. **Настроить openSUSE** для работы с доменом (если требуется безопасное монтирование).  
3. **Смонтировать папку**, указав параметры аутентификации.  

---

## **1. Настройка NFS на Windows**
### **1.1. Установка NFS-сервера**
1. Откройте **"Управление сервером"** → **"Добавить роли и компоненты"**.  
2. Выберите:  
   - **Служба NFS** → **Сервер NFS**  
3. Запустите службу **Server for NFS**.

### **1.2. Настройка общей папки**
1. Кликните правой кнопкой на папке → **Свойства** → **NFS Sharing**.  
2. Нажмите **Manage NFS Sharing** и задайте:  
   - **Anonymous UID** и **Anonymous GID** (должны соответствовать UID/GID пользователя в Linux).  
   - Или включите **Kerberos**, если нужна безопасная аутентификация.  

#### **Если не используете Kerberos (простой способ)**
В PowerShell (от админа):  
```powershell
nfsadmin server config sec=sys  # Использует UNIX-аутентификацию (UID/GID)
Set-NfsMappingStore -EnableLdapMapping $false  # Отключает AD-маппинг
Restart-Service NfsServer
```

#### **Если используете Kerberos (безопасный способ)**
```powershell
nfsadmin server config sec=krb5  # Включает Kerberos
Restart-Service NfsServer
```

---

## **2. Настройка openSUSE для работы с доменом**
### **2.1. Подключение к Active Directory (если нужно)**
```bash
sudo zypper install sssd krb5-client
sudo authselect select sssd with-mkhomedir --force
```
Настройте `/etc/sssd/sssd.conf` и `/etc/krb5.conf` для вашего домена.

### **2.2. Проверка доменного пользователя**
Узнайте `UID` и `GID` пользователя:
```bash
id username@domain
```
Пример вывода:
```
uid=10000(aduser) gid=10000(domain users) groups=...
```
(Запомните `10000:10000`)

---

## **3. Монтирование NFS в openSUSE**
### **3.1. Установка NFS-клиента**
```bash
sudo zypper install nfs-client
```

### **3.2. Монтирование (без Kerberos)**
```bash
sudo mkdir -p /mnt/nfs_win
sudo mount -t nfs <Windows_IP>:/share /mnt/nfs_win -o vers=3,nolock,uid=10000,gid=10000
```
- `uid` и `gid` должны совпадать с настройками на Windows.

### **3.3. Монтирование с Kerberos (NFSv4)**
```bash
sudo mount -t nfs4 <Windows_IP>:/share /mnt/nfs_win -o sec=krb5,vers=4.2
```
Предварительно получите Kerberos-билет:
```bash
kinit username@DOMAIN
```

### **3.4. Автомонтирование (`/etc/fstab`)**
Добавьте строку:
```bash
<Windows_IP>:/share  /mnt/nfs_win  nfs  vers=3,nolock,uid=10000,gid=10000  0  0
```
Или для Kerberos:
```bash
<Windows_IP>:/share  /mnt/nfs_win  nfs4  sec=krb5,vers=4.2  0  0
```
Примените:
```bash
sudo mount -a
```

---

## **4. Проблемы и решения**
### **Ошибка: "Permission denied"**
- Убедитесь, что `UID/GID` совпадают на Windows и Linux.  
- Проверьте, что на Windows в NFS-шаре разрешён анонимный доступ или настроен Kerberos.  

### **Ошибка: "RPC timeout"**
- Проверьте, открыты ли порты **2049/TCP** и **111/UDP** в брандмауэре Windows.  

### **Ошибка: "Stale file handle"**
```bash
sudo umount -l /mnt/nfs_win
sudo mount -a
```

---

## **Итог**
| Способ | Команда монтирования | Когда использовать |
|--------|----------------------|-------------------|
| **Без Kerberos** | `sudo mount -t nfs <IP>:/share /mnt/nfs_win -o vers=3,nolock,uid=10000,gid=10000` | Если не нужна безопасная аутентификация |
| **С Kerberos** | `sudo mount -t nfs4 <IP>:/share /mnt/nfs_win -o sec=krb5,vers=4.2` | Если требуется защищённый доступ |

Если права не работают, проверьте `id username@domain` и настройки NFS на Windows. 🚀