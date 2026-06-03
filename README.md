# 🔐 Как защитить свой VPS

> Я сменил много серверов и могу рассказать об этом.

---

## 1. Выбор провайдера

Подойдёт любой, но лучше удостовериться, что у провайдера хорошая репутация.  
Убедись, что ASN тоже принадлежит надёжному провайдеру.

---

## 2. Первые шаги после получения сервера

```bash
apt update && apt upgrade -y
```

---

## 3. Настройка SSH

Меняем порт и ужесточаем конфигурацию:

```bash
nano /etc/ssh/sshd_config
```

```ini
Port 29310                        # любой нестандартный порт

PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey

MaxAuthTries 3
LoginGraceTime 20
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
MaxSessions 5
MaxStartups 3:50:5

AllowUsers ВАШ_ЮЗЕР
```

```bash
systemctl restart ssh

# Проверяем подключение (держим старый терминал открытым!)
ssh -p 29310 -i ~/.ssh/id_rsa your_user@ip
```

---

## 4. Файрвол (UFW)

```bash
ufw default deny incoming
ufw default allow outgoing

# Открываем только нужные порты
ufw allow 29310/tcp    # новый SSH-порт

ufw enable
ufw status
```

> ⚠️ **Важно:** Перед закрытием 22-го порта держи второй терминал открытым, пока не проверишь новое подключение.

---

## 5. Fail2ban

```bash
apt install fail2ban -y
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
nano /etc/fail2ban/jail.local
```

Находим секцию `[sshd]` и настраиваем:

```ini
[sshd]
enabled  = true
port     = 29310
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 3600
findtime = 600
```

```bash
systemctl enable fail2ban
systemctl start fail2ban

# Проверяем забаненные IP
fail2ban-client status sshd
```

---

## 6. Автообновление безопасности

```bash
apt install unattended-upgrades -y
dpkg-reconfigure -plow unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Должно быть:

```ini
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

---

## 7. Блокировка root и аудит пользователей

```bash
# Блокируем пароль root
passwd -l root

# Проверяем — должно показать: root L...
passwd -S root

# Смотрим кто имеет sudo
cat /etc/sudoers
getent group sudo

# Ищем лишних пользователей (должны быть только root и наш юзер)
cat /etc/passwd | grep -v nologin | grep -v false

# Строгий umask
echo 'umask 027' >> /etc/profile
```

---

## 8. Аудит портов и сервисов

```bash
# Что слушается на портах
ss -tlnp

# Запущенные сервисы
systemctl list-units --type=service --state=running

# Отключаем лишнее
systemctl stop bluetooth
systemctl disable bluetooth

# Проверяем crontab на сторонние задачи
crontab -l
cat /etc/crontab
ls /etc/cron.*
```

---

## 9. Logwatch — ежедневные отчёты

```bash
apt install logwatch -y
logwatch --output stdout --format text --range today
```

Настройка отправки на email:

```bash
nano /etc/logwatch/conf/logwatch.conf
```

```ini
MailTo  = your@email.com
Range   = yesterday
Detail  = Low
```

---

## 10. Hardening ядра (sysctl)

```bash
nano /etc/sysctl.d/99-security.conf
```

```ini
# Защита от SYN-flood
net.ipv4.tcp_syncookies = 1

# Отключаем ICMP-редиректы
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Отключаем source routing
net.ipv4.conf.all.accept_source_route = 0

# Логируем martian-пакеты
net.ipv4.conf.all.log_martians = 1

# Защита от TIME-WAIT атак
net.ipv4.tcp_rfc1337 = 1

# ASLR — рандомизация адресного пространства
kernel.randomize_va_space = 2
```

```bash
sysctl -p /etc/sysctl.d/99-security.conf

# Проверяем
sysctl net.ipv4.tcp_syncookies
```

---

## 11. Опционально: Rootkit Hunter

```bash
apt install rkhunter mailutils -y
rkhunter --update
rkhunter --propupd

# Запускаем проверку
rkhunter --check --skip-keypress

# Смотрим только предупреждения
rkhunter --check --skip-keypress 2>&1 | grep -E 'Warning|Found'

# Добавляем в cron (каждое воскресенье в 3:00)
echo '0 3 * * 0 root rkhunter --check --skip-keypress --report-warnings-only | mail -s "rkhunter report" your@email.com' >> /etc/crontab
```

---

## 12. Опционально: AIDE — контроль целостности файлов

```bash
apt install aide aide-common -y
aideinit
cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Проверяем изменения
aide --check

# После каждого планового обновления
aide --update && cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

> AIDE запоминает хеши всех важных системных файлов и показывает любые изменения с момента baseline.

---

## Итог

Эти шаги дадут практически полную защиту VPS от внешних атак.  
Но помни — есть и другие векторы, например **социальная инженерия**.

Базовые правила:
- Не светить IP без необходимости
- Не давать доступ сторонним лицам
- Регулярно проверять логи

---

*author: [@eirse](https://t.me/eirse)*
