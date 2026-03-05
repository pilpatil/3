## 1. Импорт пользователей на сервере BR-SRV
Монтируем образ с данными:
mount /dev/sr0 /mnt/
Создаем скрипт для импорта пользователей:
nano /var/import.sh
**Содержимое `/var/import.sh` (исправлены опечатки и кавычки):**
#!/bin/bash
CSV_FILE="/mnt/Users.csv"
DOMAIN="AU-TEAM.IRPO"
ADMIN_USER="Administrator"
ADMIN_PASS="P@ssw0rd"
while IFS=';' read -r fname lname role phone ou street zip city country password; do
if [[ "$fname" == "First Name" ]]; then
continue
fi
username=$(echo "${fname:0:1}${lname}" | tr '[:upper:]' '[:lower:]')
sudo samba-tool ou create "OU=${ou},DC=AU-TEAM,DC=IRPO" --description="${ou} department"
echo "Adding user: $username in OU=$ou"
sudo samba-tool user add "$username" "$password" \
--given-name="$fname" \
--surname="$lname" \
--job-title="$role" \
--telephone-number="$phone" \
--userou="OU=$ou"
done < "${CSV_FILE}"
echo "Complete import"
Делаем скрипт исполняемым и запускаем:
chmod +x /var/import.sh
/var/import.sh
_Проверка: через утилиту ADMC проверяем появление пользователей и OU._

## 3. Настройка IPsec (Шифрование поверх GRE)
На обеих маршрутизаторах (HQ-RTR и BR-RTR) редактируем файл `ipsec.conf`:
epmi strongswan
nano /etc/strongswan/ipsec.conf
Добавляем:
conn gre
type=tunnel
authby=secret
left=10.5.5.1
right=10.5.5.2
leftprotoport=gre
rightprotoport=gre
auto=start
pfs=no
IP-адреса left и right зеркально меняются на противоположном роутере
На обеих маршрутизаторах редактируем файл с паролями:
nano /etc/strongswan/ipsec.secrets
Добавляем строку:
10.5.5.1 10.5.5.2 :PSK "P@ssw0rd"
Включаем и запускаем службу:
systemctl enable --now strongswan-starter.service
Проверка на роутере:
tcpdump -i ens18 -n -p esp

## 4. Настройка межсетевого экрана (nftables) на HQ-RTR и BR-RTR
nano /etc/nftables/nftables.nft
Приводим таблицу фильтров к такому виду:
table inet filter {
chain input {
type filter hook input priority 0; policy drop;
log prefix "Dropped Input: " level debug
iif lo accept
ct state established,related accept
tcp dport { 22,514,53,80,443,3015,445,139,88,2026,8080,2049,389 } accept
udp dport { 53,123,500,4500,88,137,8080,2049 } accept
ip protocol icmp accept
ip protocol esp accept
ip protocol gre accept
ip protocol ospf accept
}
chain forward {
type filter hook forward priority 0; policy drop;
log prefix "Dropped forward: " level debug
iif lo accept
ct state established,related accept
tcp dport { 22,514,53,80,443,3015,445,139,88,2026,8080,2049,389 } accept
udp dport { 53,123,500,4500,88,137,8080,2049 } accept
ip protocol icmp accept
ip protocol esp accept
ip protocol gre accept
ip protocol ospf accept
}
chain output {
type filter hook output priority 0; policy accept;
}
}
НА BR-RTR ДЕЛАЕМ АНАЛОГИЧНО

## 5. Настройка принт-сервера CUPS на HQ-SRV
nano /etc/cups/cupsd.conf
- Заменить `Listen localhost:631` на `Listen hq-srv.au-team.irpo:631`
- В блоках `<Location />`, `<Location /admin>` и `<Location /admin/conf>` закомментировать `#Order allow,deny` и вписать `Allow any`
Перезагрузка службы:
systemctl restart cups
Настройка клиента:
- Зайти по адресу `https://hq-srv.au-team.irpo:631`.
- Авторизоваться как `root`.
- Добавить виртуальный PDF принтер (CUPS-PDF), поставив галочку "Разрешить совместный доступ".
- В пункте "Создать" выбираем "Generic", "Модель" выбираем "Generic CUPS-PDF"
- На клиентской ОС добавить найденный сетевой принтер через утилиту "Принтеры", вписав имя сервера hq-srv.

## 6. Логирование (Rsyslog) на HQ-RTR, BR-RTR, BR-SRV
Настройка Сервера логов (HQ-SRV):
nano /etc/rsyslog.d/00_common.conf
module(load="imjournal")
module(load="imuxsock")
module(load="imtcp")
input(type="imtcp" port="514")
if $fromhost-ip contains '192.168.0.62' then {
*.warn /opt/hq-rtr/hq-rtr.log
}
if $fromhost-ip contains '10.5.5.2' then {
*.warn /opt/br-rtr/br-rtr.log
}
if $fromhost-ip contains '192.168.1.1' then {
*.warn /opt/br-srv/br-srv.log
}
Создаем каталоги логов и перезапускаем службу:
mkdir -p /opt/hq-rtr /opt/br-rtr /opt/br-srv
systemctl enable --now rsyslog
Настройка Клиентов (HQ-RTR, BR-RTR, BR-SRV):
В файле `/etc/rsyslog.d/00_common.conf` раскомментировать модули `imjournal` и `imuxsock` и добавить:
*.warn @@192.168.0.1:514
Перезапуск:
systemctl enable --now rsyslog
Настройка Ротации логов (на сервере):
nano /etc/logrotate.conf
Добавить в конец файла:
/opt/hq-rtr/*.log
/opt/br-rtr/*.log
/opt/br-srv/*.log
{
minsize 10M
rotate 4
weekly
compress
missingok
}
_
Примечание: logrotate в systemd запускается через таймер, поэтому правильная команда для автозапуска:_
systemctl enable --now logrotate.timer
logrotate -d /etc/logrotate.conf

## 7. Мониторинг (Prometheus + Grafana)
Запуск node
exporter на клиентских машинах:
_
systemctl enable --now prometheus-node_exporter
Настройка Prometheus на HQ-SRV:
nano /etc/prometheus/prometheus.yml
В блоке `static_configs` приводим к виду (одинарные прямые кавычки!):
static_configs:
- targets: ['localhost:9090', 'hq-srv:9100', 'br-srv:9100']
Запуск служб:
systemctl enable --now grafana-server
systemctl enable --now prometheus
systemctl enable --now prometheus-node_exporter
Настройка Grafana:
- Заходим на `http://hq-srv:3000` (admin/admin),
- меняем пароль на P@ssw0rd
- Заходим в профиль пользователя (правый верхний угол с авой рандомной), "Profile" и меняем язык на русский.
- В меню "Подключения" в пункте "Источники данных" нажимаем "Добавить источник данных"
- Выбираем Prometheus, адрес сборщика задаем https://localhost:9090 (Server URL)
- В меню "Дашборды" создаём дашборд.
- Выбираем импорт дашборка и указываем 11074.
- Выбираем службу для дашборда Prometheus. Нажимаем Импорт.
- Заходим в меню на 3 точки, выбираем редактировать и переименовываем заголовок на "Информация по серверам"

## 8. Инвентаризация через Ansible
mount /dev/sr0 /mnt/
cp /mnt/playbook/get_hostname_address.yml /etc/ansible/
chmod +x /etc/ansible/get_hostname_address.yml
Редактируем Playbook (строго соблюдаем отступы):
nano /etc/ansible/get_hostname_address.yml
- name: Инвентаризация
hosts: HQ-SRV, HQ-CLI
tasks:
- name: Получение данных с хоста
copy:
dest: "/etc/ansible/PC-INFO/{{ ansible_hostname }}.yml"
content: |
Hostname: {{ ansible_hostname }}
IP_Address: {{ ansible_default_ipv4.address }}
delegate_to: localhost
Создаем директорию и запускаем:
mkdir -p /etc/ansible/PC-INFO
ansible-playbook /etc/ansible/get_hostname_address.yml
Проверка:
cat /etc/ansible/PC-INFO/hq-srv.yml

## 9. Защита SSH с помощью Fail2ban
Редактируем файл конфигурации (можно создать `/etc/fail2ban/jail.local` или править `jail.conf`):
nano /etc/fail2ban/jail.conf
Находим секцию `[sshd]` и приводим к виду:
[sshd]
enabled = true
port = 2026
logpath = /var/log/auth.log
backend = systemd
filter = sshd
action = nftables[name=SSH, port=2026, protocol=tcp]
maxretry = 2
bantime = 1m
Включаем службу:
systemctl enable --now fail2ban
Проверка статуса после попыток неправильного ввода пароля по ssh:
fail2ban-client status sshd
