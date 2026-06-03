Как защитить свой VPS?

Я сменил много серверов и могу рассказать об этом.

Начинаем с выбора провайдера. На самом деле подойдет любой, но лучше удостовериться, что у провайдера хорошая репутация. Выбираем сервер, убеждаемся что asn принадлежит так же хорошему провайдеру.

Как только мы получили наш сервер — Обновляемся apt update && apt upgrade -y. Далее меняем порт на ссш (например 29310). /etc/ssh/sshd_config. Запускаем файрвол смотрит что он работает и открываем нужный порт. Тут же в конфиге запрещаем рут логин PermitRootLogin no, так же PasswordAuthentication no, PubkeyAuthentication yes, AuthenticationMethods publickey. MaxAuthTries 3, LoginGraceTime 20, ClientAliveInterval 300,  X11Forwarding no, MaxSessions 5,
MaxStartups 3:50:5, ClientAliveCountMax 2, AllowUsers "ваш юзер", перезапускаем ссш и проверяем systemctl restart ssh, 
ssh -p 'новый порт' -i ~/.ssh/id_rsa your_user@ip. 

Файровл - ufw default deny incoming, ufw default allow outgoing. Открываем нужные порты только под нужные службы, остальные порты должны быть закрыты (22 тоже).  Важно: Перед закрытием 22 порта, держим второй терминал открытым пока не проверим новое подключение.

Ставим fail2ban. Далее делаем локальный конфиг cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local. в нем пишем # Найти [sshd] и настроить:
[sshd]
enabled = true
port = (ваш ссш)
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
Далее systemctl enable fail2ban и systemctl start fail2ban
Можно проверить что все работает fail2ban-client status sshd | grep 'забаненый айпи'

Включаем автообновление безопасности apt install unattended-upgrades -y и dpkg-reconfigure -plow unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades
Должно быть
# APT::Periodic::Update-Package-Lists "1";
# APT::Periodic::Unattended-Upgrade "1";


Заблокируем пароль root
passwd -l root
Проверим
passwd -S root
Должно показать: root L..
Проверим кто имеет sudo
cat /etc/sudoers
getent group sudo
Должен быть только основной юзер
Ищем лишних юзеров
cat /etc/passwd | grep -v nologin | grep -v false
Должны быть только root и наш юзер. Остальное — подозрительно
Установить строгий umask
echo 'umask 027' >> /etc/profile

Смотрим что слушается на портах ss -tlnp и смотрим запущенные сервисы systemctl list-units --type=service --state=running,  ничего лишнего не должно быть. Можно отключить лишнее systemctl stop bluetooth
systemctl disable bluetooth
Проверяем кронтаб на сторонние задачи. crontab -l , cat /etc/crontab , ls /etc/cron.*


Ставим logwatch для ежедневных отчётов
apt install logwatch -y , logwatch --output stdout --format text --range today
Можно сдедлать отправку логов на email 
В /etc/logwatch/conf/logwatch.conf:
MailTo = ИМЕЙЛ
Range = yesterday
Detail = Low

Создаем nano /etc/sysctl.d/99-security.conf
Тут пишем 
net.ipv4.tcp_syncookies = 1

net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.tcp_rfc1337 = 1
kernel.randomize_va_space = 2

Далее sysctl -p /etc/sysctl.d/99-security.conf
Смотрим sysctl net.ipv4.tcp_syncookies


Опционально:
apt install rkhunter -y
rkhunter --update
apt install mailutils -y

rkhunter --propupd
Запустить проверку
rkhunter --check --skip-keypress

rkhunter --check --skip-keypress 2>&1 | grep -E 'Warning|Found'

echo '0 3 * * 0 root rkhunter --check --skip-keypress --report-warnings-only | mail -s "rkhunter report" ИМЕЙЛ' >> /etc/crontab

Опционально:
apt install aide aide-common -y
aideinit
cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
AIDE запоминает хеши всех важных системных файлов
Проверим изменения
aide --check
Покажет все изменённые файлы с момента baseline
Запускать после каждого планового обновления: aide --update && cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db

Это даст практически полную защиту впс, но помните, что есть и другие виды атак, например, как социальная инженерия. Поэтому помните о базовых правилах защиты — не показывать айпи/не давать доступ сторонним лицам и тд.
author @eirse tg
