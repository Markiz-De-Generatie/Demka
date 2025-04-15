




## Выполнение
### 1. Произведите базовую настройку устройств

- Настройте имена устройств согласно топологии. Используйте
полное доменное имя

Для быстрой смены имени хоста нужно ввести соответствущую команду и перелогиниться.
 
``` bash
hostnamectl hostname ISP;newgrp
```


## Схема сети
![Network_Scheme](https://github.com/user-attachments/assets/975b24d0-d734-4695-b3b1-154282d90b6c)



## Адресация устройств

| Имя устройства | IP - Адрес  | Интерфейс | Шлюз по умолчанию | 
|:---------:|:--------------------------:|:--------------------------:|:--------------------:|
|     ISP     |           192.168.0.50/24         |   ens18   |  192.168.0.1        |        
|     ISP     |           172.16.4.1/28          |   ens19 |    172.16.4.2       |        
|     ISP     |           172.16.5.1/28           |   ens20  |   172.16.5.2         |        
|     HQ-CLI     |           172.16.200.10/28            |   ens18   |  172.16.200.1        |  
|     HQ-SRV  |           172.16.100.10/26            |   ens18    | 172.16.100.1       |        
|     BR-SRV     |           172.16.50.10/27          |    ens18   | 172.16.50.1       |  
|     HQ-RTR     |           172.16.4.2/28          |    ens18   | 172.16.4.1       |  
|     HQ-RTR     |           172.16.100.3/27          |    ens19   | 172.16.100.1       |  
|     HQ-RTR     |           172.16.200.3/27          |    ens19   | 172.16.200.1       |   
|     BR-RTR    |           172.16.5.2/28          |    ens18   | 172.16.50.1       |  
|     BR-RTR    |           172.16.50.1/27          |    ens19   | 172.16.50.1       |  
| MGMT VLAN     |      172.16.99.0/29                |            |                  | 


- Настройка IP-адресации на хостах проводится в файле /etc/network/interafaces

  Пример настроенного файла на ISP:
``` bash
  # This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug ens18
iface ens18 inet dhcp

auto ens19
iface ens19 inet static
address 172.16.4.1
netmask 255.255.255.240

auto ens20
iface ens20 inet static
address 172.16.5.1
netmask 255.255.255.240
```
Также нужно не забыть добавить машрут по умолчанию:
```bash
ip route add default via <IP gate>
```
Для того чтобы иметь возможность устанавливать пакеты нужно привести файл /etc/apt/sources.list на всех машинах к следующему виду:

``` bash
deb https://deb.debian.org/debian bookworm main non-free-firmware
deb-src https://deb.debian.org/debian bookworm main non-free-firmware

deb https://deb.debian.org/debian bookworm-updates main non-free-firmware
deb-src https://deb.debian.org/debian bookworm-updates main non-free-firmware
```

При настроенной сетевой связности,заполнить sources.list можно следующим образом:
``` bash
cat /etc/apt/sources.list | ssh locadm@172.16.4.2 'cat >> /home/locadm/list.txt'
```

Для того чтобы работала машрутизация на устройствах ISP, HQ-RTR, BR-RTR нужно включить параметр ядра отвечающий за маршрутизацию IP пакетов в файле /etc/sysctl.conf
``` bash
net.ipv4.ip_forward=1
```
Для применения параметров:
```bash
sysctl -p
```

### 2. Настройка ISP

#### iptables

Установка необходимых пакетов для iptables:
``` bash
apt install iptables iptables-persistent
```
Создание правил iptables на ISP
``` bash
iptables –t nat –A POSTROUTING –s 172.16.4.0/28 –o ens18 –j MASQUERADE  
iptables –t nat –A POSTROUTING –s 172.16.5.0/28 –o ens18 –j MASQUERADE  
iptables-save > /etc/iptables/rules.v4
```
В опции -o должен быть именно out-interface выходящий во внешнюю сеть (прокололся на этом)

### 3. Настройка HQ-RTR и BR-RTR
Настройка Inter VLAN routing, для этого необходимо установить следующие пакеты и подгрузить следующие модули:
``` bash
apt install vlan
modprobe 8021q
echo 8021q >> /etc/modules
```
Файл /etc/network/interfaces для HQ-RTR
``` bash
auto ens18
iface ens18 inet static
address 172.16.4.2
netmask 255.255.255.240
gateway 172.16.4.1

auto ens19
iface ens19 inet static
address 172.16.100.1
netmask 255.255.255.192

auto ens19:1
iface ens19:1 inet static
address 172.16.200.1
netmask 255.255.255.240

auto ens19.100
iface ens19.100 inet static
address 172.16.100.3
netmask 255.255.255.192
vlan-raw-device ens19

auto ens19.200
iface ens19.200 inet static
address 172.16.200.3
netmask 255.255.255.240
vlan-raw-device ens19:1
```
Создание правил iptables на HQ-RTR 
``` bash
iptables –t nat –A POSTROUTING –s 172.16.100.0/26 –o ens18 –j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.200.0/28 -o ens18 -j MASQUERADE
iptables-save > /etc/iptables/rules.v4
```
Создание правил iptables на BR-RTR
``` bash
iptables -t nat -A POSTROUTING -s 172.16.50.0/27 -o ens18 -j MASQUERADE
iptables-save > /etc/iptables/rules.v4
```

### 4. Создание локальных учетных записей
Добавление пользователей на HQ-SRV и BR-SRV:
``` bash
useradd -m -d /home/sshuser -s /bin/bash -u 1010 sshuser
passwd sshuser
```
Запуск sudo без дополнительной аутентификации:
visudo /etc/sudoers
``` bash
sshuser ALL=(ALL:ALL) NOPASSWD: ALL
```
Добавление пользователей на HQ-RTR и BR-RTR:
``` bash
useradd -m -d /home/net_admin -s /bin/bash net_admin
passwd net_admin
```
Запуск sudo без дополнительной аутентификации:
``` bash
net_admin ALL=(ALL:ALL) NOPASSWD: ALL
```

### 5. Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV:
В файле /etc/ssh/sshd_config меняется порт на 2024 и добавляются опции для разрешения подключения только одного пользователя и других опций:
``` bash
Port 2024
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh-banner
```
Содержимое /etc/ssh-banner
```
Authorized access only
``` 
### 6. Конфигурация IP тунеля между HQ и BR офисами:

Необходимо подгрузить модуль для работы с GRE:
``` bash
echo "ip_gre" | tee -a /etc/modules
```
Далее необходимо добавить дополнительный виртуальный интерфейс посредством конфигурации файла /etc/network/interfaces:
Пример файла на HQ-RTR
``` bash
auto tun1
iface tun1 inet tunnel
address 10.10.0.1
netmask 255.255.255.252
mode gre
local 172.16.4.2
endpoint 172.16.5.2
ttl 64
```
Пример файла на BR-RTR
``` bash
auto tun1
iface tun1 inet tunnel
address 10.10.0.2
netmask 255.255.255.252
mode gre
local 172.16.5.2
endpoint 172.16.4.2
ttl 64
```
### 7. Обеспечение динамической маршрутизации
Установка пакета frr
``` bash
apt install frr
```
Включение протокола динамической маршрутизации OSPF
``` bash
nano /etc/frr/daemons
ospfd=yes
```
Настройка ospf на HQ-RTR:
``` bash
router osfp
#Отключение hello пакетов на всех интерфейсах
passive-interface default
network 172.16.200.0/26 area 1
network 172.16.100.0/28 area 1
network 10.10.0.0/30 area 0
#Необязательная комадна, используется для совместимости
area 0 authentication
interface tun1
no ip ospf passive
ip ospf authentication
ip ospf authentication-key password
```
Настройка ospf на BR-RTR:
``` bash
router osfp
#Отключение hello пакетов на всех интерфейсах
passive-interface default
network 172.16.50.0/27 area 2
network 10.10.0.0/30 area 0
#Необязательная комадна, используется для совместимости
area 0 authentication
interface tun1
no ip ospf passive
ip ospf authentication
ip ospf authentication-key password
```
### 8. Настройка автоматического распределения IP-адресов на HQ-RTR
Установка isc-dhcp-server 
```
apt install isc-dhcp-server
```
В файле /etc/default/isc-dhcp-server 
``` bash
INTERFACES="ens19:1"
```
Содержимое файла /etc/dhcp/dhcpd.conf
``` bash
ddns-update-style interim;
authoritative;
subnet 172.16.200.0 netmask 255.255.255.240 {
       range 172.16.200.4 172.16.200.14;
       option domain-name-servers  172.16.100.10;
       option domain-name "au-team.irpo";
       default-lease-time 600;
       max-lease-time 7200;
}
```
На HQ-CLI в /etc/network/interfaces:
```bash
auto ens18
iface ens18 inet dhcp
```

### 9. Настройка DNS сервера на HQ-SRV
Устанавливаем bind и утилиты для преобразования имен:
``` bash
apt install bind9 dnsutils
```
Конфигурация файла /etc/bind/named.conf.options, в forwarders указываем DNS сервер для пересылки запросов:
``` bash
forwarders {
      8.8.8.8;
}
listen-on {any;};
allow-recursion { 172.16.100.0/26; 172.16.50.0/27; 172.16.200.0/28; 10.10.0.0/30;};
listen-on-v6 { none; };
```
Копируем стандартные файлы зон 
``` bash
cp /etc/bind/db.local /etc/bind/au-team.irpo
cp /etc/bind/db.127   /etc/bind/au-team.reverse
```
Прямая зона DNS:

![image](https://github.com/user-attachments/assets/192586d5-4a93-4e58-9c7b-8817e1d300c4)

Обратная зона DNS:

![image](https://github.com/user-attachments/assets/aae2df3b-be36-4454-a867-84d825a83cd1)

Файл /etc/bind/named.conf.default-zones, в нём прописываем все имеющиеся зоны:

![image](https://github.com/user-attachments/assets/9ec5e6e3-fb46-436a-9f7e-c6d03cdbc7ba)

### 10. Настройка даты и времени согласно месту проведения экзамена

Установка часового пояса выполняется с помощью следущей команды:
``` bash
timedatectl set-timezone Asia/Tomsk
```






 


