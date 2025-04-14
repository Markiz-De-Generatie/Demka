




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






