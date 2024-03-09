# Решение конкурсного задания регионального чемпионата "Профессионалы-2024" Санкт-Петербург.
## Модуль Б. Настройка технических и программных средств информационно-коммуникационных систем.

### 1. **Базовая настройка** 


### 2. **Настройка ISP**

#### Базовая настройка, адресация и маршрутизация

Задаём имя устройству:

``` bash
hostnamectl set-hostname ISP;exec bash
```

>[!NOTE]
>Для каждого сетевого интерфейса необходимо создать директорию по пути /etc/net/ifaces/<NAME_INTERFACE>/options:
>[!NOTE]
>enp0s3 – WAN, остальные интерфейсы – по MAC-адресу.

``` bash
mkdir /etc/net/ifaces/enp0s{8..9}
```

Настройка интерфейса enp0s3, смотрящего в сеть:

``` bash
nano /etc/net/ifaces/enp0s3/options
```
``` bash
TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=dhcp
IPV4_CONFIG=yes
```

Настройка интерфейса enp0s8:

``` bash
nano /etc/net/ifaces/enp0s8/options
```
``` bash
TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
IPV4_CONFIG=yes
```

Скопировать в соответствующие директории: 

``` bash
cp /etc/net/ifaces/enp0s8/options /etc/net/ifaces/enp0s9/
```

Теперь назначаем IPv4-адреса на сетевые интерфейсы:

``` bash
echo 11.11.11.1/24 > /etc/net/ifaces/enp0s8/ipv4address 
```
``` bash
echo 22.22.22.1/24 > /etc/net/ifaces/enp0s9/ipv4address
```

Включаем forwarding – маршрутизацию:

``` bash
echo net.ipv4.ip_forward=1 >> /etc/net/sysctl.conf -p
```

Для применения всех сетевых настроек перезагружаем службу “network”

``` bash
systemctl restart network
```

#### Настройка динамической трансляции адресов
Устанавливаем nftables:

``` bash
apt-get update && apt-get install -y nftables
```

Включаем и добавляем в автозагрузку службу nftables:

``` bash
systemctl enable --now nftables
```

Далее создаём необходимую структуру для nftables (семейство, таблица, цепочка) для настройки NAT:
``` bash
nft add table ip nat
```
``` bash
nft add chain ip nat postrouting '{ type nat hook postrouting priority 0; }'
```
``` bash
nft add rule ip nat postrouting ip saddr 11.11.11.0/24 oifname "enp0s3" counter masquerade
```
``` bash
nft add rule ip nat postrouting ip saddr 22.22.22.0/24 oifname "enp0s3" counter masquerade
``` 
Сохраняем правила nftables:
>[!NOTE]
>Так как в конфигурационном файле /etc/nftables/nftables.nft уже есть информация о таблице filter – необходимо дописать только что созданную информацию о таблице nat, запишем в конфигурационный файл /etc/nftables/nftables.nft последние 8 строк (от 14 до 21) вывода команды nft list ruleset:
``` bash
nft list ruleset | tail -n8 | tee -a /etc/nftables/nftables.nft
```

Перезагружаем службу nftables:

``` bash
systemctl restart nftables
```

Смотрим правила:
``` bash
nft list ruleset
```

#### Настройка протокола динамической конфигурации хостов (опционально)
Установка DHCP:

``` bash
apt-get install -y dhcp-server
```

Укажем сетевой интерефейс, через который будет работать DHCP-сервер:

``` bash
sed -i "s/DHCPDARGS=/DHCPDARGS=ens35 ens36 ens37/g" /etc/sysconfig/dhcpd
```

После чего, необходимо привести файл “/etc/dhcp/dhcpd.conf” к следующему виду:
>[!NOTE]
>Если планируется далее развёртывать и DNS-сервер, то выдаём в качестве option domain-name-servers соответствующие адреса;
>В противном случае, можно расскомментировать строку “option domain-name-servers 77.88.8.8 77.88.8.88;“

``` bash
nano /etc/dhcp/dhcpd.conf
```
``` bash
# dhcpd.conf
# option domain-name-servers 77.88.8.8, 77.88.8.88;

default-lease-time 600;
max-lease-time 7200;

# WAN
subnet 192.168.1.0 netmask 255.255.255.0 {}

# INET1
subnet 11.11.11.0 netmask 255.255.255.0 {
    range 11.11.11.100 11.11.11.200;
    option domain-name-servers 11.11.11.1;
    option routers 11.11.11.1;
}

# INET2
subnet 22.22.22.0 netmask 255.255.255.0 {
    range 22.22.22.100 22.22.22.200;
    option domain-name-servers 22.22.22.1;
    option routers 22.22.22.1;
}
```
 
Проверим конфигурацию файла /etc/dhcp/dhcpd.conf:

``` bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

Включаем и добавляем в автозагрузку службу dhcpd:

``` bash
systemctl enable --now dhcpd
```

Проверяем работоспособность службы dhcpd:

``` bash
systemctl status dhcpd
```

#### Настройка DNS для доступа в Интернет (опционально)
Установка DNS

``` bash
apt-get install bind -y
```

Редактируем конфигурационный файл по пути “/var/lib/bind/etc/options.conf”:

``` bash
nano /var/lib/bind/etc/options.conf
```
``` bash
listen-on { any; };
allow-query { any; };
forwarders { 8.8.8.8; };
recursion yes;
```

При помощи утилиты “named-checkconf” проверяем наличие ошибок в конфигурационном файле, если результат выполнения данной команды пуст – значит ошибок нет.

``` bash
named-checkconf -z
```

Для указания информации о DNS – сервере, можно добавить запись в файл “/etc/resolv.conf”:

``` bash
nano /etc/resolv.conf
```
``` bash
search company.prof
domain company.prof
nameserver 8.8.8.8
```

Выполняем запуск и добавление в автозагрузку DNS – сервера:

``` bash
systemctl enable --now bind
```

### 4. **Настройка дисковой подсистемы**

#### Настройка RAID на RTR1
Обновим пакеты и установим cfdisk:

``` bash
apt-get update && apt-get install cfdisk nano
```

Просмотр дисковой подсистемы:

``` bash
lsblk
```

На новых неразмеченных дисках необходимо создать разделы при помощи утилиты cfdisk:

``` bash
cfdisk /dev/sdb
```
``` bash
cfdisk /dev/sdc
```
``` bash
cfdisk /dev/sdd
```

#### делаем разметку диска
Выбираем в меню нужные параметры:

1. gpt
2. Новый
3. Полный размер
4. Тип – Linux RAID
5. Запись
6. yes


Далее необходимо собрать RAID5-массив:

``` bash
mdadm --create /dev/md1 --level=5 --raid-devices=3 /dev/sdb1 /dev/sdc1 /dev/sdd1
```

После того, как массив собран, его необходимо сохранить:

``` bash
mdadm --detail --scan --verbose | tee -a /etc/mdadm.conf
```

Форматируем в ext4:

``` bash
mkfs.ext4 /dev/md1
```

Создаём точку монтирования:

``` bash
mkdir /opt/data
```

Добавляем запись в файл /etc/fstab (используем tab вместо пробелов):

``` bash
nano /etc/fstab
 ```
``` bash
/dev/md1    /opt/data   ext4    defaults    0      0
```

Применяем монтирование и проверяем результат:

``` bash
mount -av
```
``` bash
lsblk
```

#### Настройка LVM на RTR2:
Обновим пакеты и установим lvm2:

``` bash
apt-get update && apt-get install -y nano lvm2
```

Просмотр дисковой подсистемы:

``` bash
lsblk
```

Помечаем диски, что они будут использоваться для LVM:

``` bash
pvcreate /dev/sdb /dev/sdc
```

Инициализированные на первом этапе диски должны быть объединены в группы.

``` bash
vgcreate VG /dev/sdb /dev/sdc
```

Последний этап — создание логического раздела из группы томов

``` bash
lvcreate -l 100%FREE -n DATA VG
```

Создадим файловую систему ext4:

``` bash
mkfs.ext4 /dev/VG/DATA
```

Создаём точку монтирования:

``` bash
mkdir /opt/data
```

Добавляем запись в файл /etc/fstab (используем tab вместо пробелов):

``` bash
nano /etc/fstab
```
``` bash
/dev/VG/DATA    /opt/data    ext4    defaults    1    2
```

Применяем монтирование и проверяем результат:
``` bash
mount -av
```
``` bash
lsblk
```

### 5. **Установка и настройка сервера баз данных**
#### Установка MariaDB
Обновляем пакеты и устанавливаем wget:

``` bash
apt-get update && apt-get install wget
```

Устанавливаем MariaDB:

``` bash
apt-get install -y mariadb-server
```

Запускаем и добавляем в автозагрузку:

``` bash
systemctl enable --now mariadb
```

Задаем пароль root:

``` bash
mysqladmin -uroot password
``` 

Заходим в MariaDB

``` bash
mariadb -p
``` 
``` bash
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'P@ssw0rd' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

Разрешаем доступ из сети

``` bash
sed -i "s/skip-networking/#skip-networking/g" /etc/my.cnf.d/server.cnf
```
``` bash
mariadb -p
``` 
``` bash
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'P@ssw0rd'; 
EXIT;
systemctl restart mariadb
```

#### Установка и настройка Adminer:
Установим php:

``` bash
apt-get install -y php8.2 php8.2-{fpm-fcgi,mbstring,zip,gd,libs,mysqlnd,mysqlnd-mysqli,bz2,curl,mcrypt,opcache,openssl} libmcrypt
```

Запускаем и добавляем в автозагрузку:

``` bash
systemctl enable --now php8.2-fpm.service
```

Установим apache2:

``` bash
apt-get install -y httpd2 apache2-{httpd-prefork,mod_php8.2}
```

Запускаем и добавляем в автозагрузку:

``` bash
systemctl enable --now httpd2.service
```

Создаем и переходим в директорию:

``` bash
mkdir /var/www/html/adminer/
```
``` bash
cd /var/www/html/adminer/
```

Установим adminer с помощью wget:
``` bash
wget -O index.php https://github.com/vrana/adminer/releases/download/v4.8.1/adminer-4.8.1-en.php
```
``` bash
wget https://github.com/vrana/adminer/releases/download/v4.8.1/adminer-4.8.1-en.php
```
Назначаем владельца:
``` bash
chown -R apache:apache index.php /var/www/html/adminer/
```
``` bash
chmod -R 775 /var/www/html/adminer/
```

Перезагружаем службу:

``` bash
systemctl restart httpd2.service
```

Создаем пользователя adminer:

``` bash
mariadb -p
```
``` bash 
CREATE DATABASE adminer;
CREATE USER 'adminer'@'localhost' IDENTIFIED BY 'Password';
GRANT ALL ON adminer .* TO 'adminer'@'localhost';
```


### 6. **Настройка системы централизованного журналирования**
#### Установка Rsyslog
Обновить и установить пакеты Rsyslog:

``` bash
apt-get update && apt-get install -y rsyslog rsyslog-mysql
```

Добавить строки:

``` bash
nano /usr/share/doc/rsyslog-mysql-8.2304.0/createDB.sql
```
``` bash
CREATE DATABASE Syslog;
USE Syslog;
```
 
Раскомментировать module udp и tcp и добавить строки:
``` bash
nano /etc/rsyslog.d/00_common.conf
```
``` bash
$template RemoteLogs,”/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log”
*.* ?RemoteLogs
& ~
```

Включить и добавить в автозагрузку:

``` bash
systemctl enable --now rsyslog
```

Создаем базу данных:

``` bash
mariadb -p < /usr/share/doc/rsyslog-mysql-8.2304.0/createDB.sql
```

Создаем пользователя:

``` bash
mariadb -p
```
``` bash
GRANT ALL PRIVILEGES ON Sysloga.* TO 'arsyslog'@'localhost' IDENTIFIED BY 'rsyspwd';
FLUSH PRIVILEGES;
EXIT;
```

Перезагружаем службу:

``` bash
systemctl restart rsyslog
```

#### Установка loganalyzer:
Установим пакеты с помощью wget:

``` bash
wget http://download.adiscon.com/loganalyzer/loganalyzer-4.1.13.tar.gz
```

Распакуем архив:

``` bash
tar xzf loganalyzer-4.1.13.tar.gz
```

Создаем и переносим, переходим в директорию:

``` bash
mkdir /var/www/html/loganalyzer
```
``` bash
mv loganalyzer-4.1.13/src /var/www/html/loganalyzer
```
``` bash
cd /var/www/html/loganalyzer/src
```

Создаем файл:

``` bash
touch config.php
```

Назначаем пользователя:

``` bash
chmod 777 config.php
```

Переносим в нужную директорию:

``` bash
mv /var/www/html/loganalyzer/src/* /var/www/html/loganalyzer
```

Перезагружаем службу:

``` bash
systemctl restart rsyslog
```

### 7. **Настройка системы централизованного мониторинга**
### Установка и настройка Prometheus.
Обновим пакеты и установим prometheus:

``` bash
apt-get update && apt-get install prometheus
```

Включаем prometheus и добавляем в автозагрузку:

``` bash
systemctl enable --now prometheus.service
```

Уставим с помощью wget сборщик пакетов node_exporter:

``` bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
```
``` bash
tar -xvzf node_exporter-1.5.0.linux-amd64.tar.gz
```

Настройка сервиса node_exporter:

``` bash
useradd -rs /bin/false nodeusr
```
``` bash
mv node_exporter-1.5.0.linux-amd64/node_exporter /usr/local/bin/
```

Напишем конфигурационный файл сервиса:

``` bash
nano /etc/systemd/system/node_exporter.service
```
``` bash
[Unit]
Description=Node Exporter
After=network.target
[Service]
User=nodeusr
Group=nodeusr
Type=simple
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target
```
 
Включаем node_exporter и добавляем в автозагрузку, проверяем статус:

``` bash
systemctl enable node_exporter --now
```
``` bash
systemctl status node_exporter
```

Правим конфигурационный файл prometheus:

``` bash
nano /etc/prometheus/prometheus.yml
 ```
``` bash
scrape_configs:
    – job_name: ‘node-exporter’
    static_configs:
    – targets: [‘localhost:9100’, ‘IP_SRV1:9100’, ‘IP_SRV2:9100’, ‘IP_RTR2:9100’]
```

Перезагружаем prometheus и node_exporter:

``` bash
systemctl restart prometheus
```
``` bash
systemctl restart node_exporter
```

#### Установка и настройка Grafana.
Обновим пакеты и установим grafana:

``` bash
apt-get update && apt-get install grafana
```

Включаем prometheus и добавляем в автозагрузку:

``` bash
systemctl enable --now grafana-server.service
```
