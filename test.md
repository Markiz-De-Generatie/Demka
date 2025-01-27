

## Схема сети 
![Demo2025](https://github.com/user-attachments/assets/a5cfcdd1-d9ce-4e34-b44b-d2630219103a)


## Выполнение
### 1. Произведите базовую настройку устройств

- Настройте имена устройств согласно топологии. Используйте
полное доменное имя

На машине с Alt JeOS:

Для быстрой смены имени хоста нужно ввести соответствущую команду и перелогиниться.
На JeOS нет встроенной команды newgrp, поэтому просто выходим из оболочки. 

``` bash
hostnamectl hostname ISP
exit или logout
```
На остальных машинах с Alt Linux:

```  bash
hostnamectl hostname HQ-SRV
newgrp
```
На EcoRouter:
```
hostname HQ-RTR

```

Не забываем прописать write memory после проведённых настроек!

- На всех устройствах необходимо сконфигурировать IPv4

  На машинах с Alt Linux необходима конфигурация файлов /etc/net/ifaces/<имя интерфейса>/ options, ipv4address, ipv4route
  Для настройки статического адреса приводим файл options к следующему виду:

  ``` bash
  NM_CONTROLLED=no
  SYSTEMD_CONTROLLED=no
  DISABLED=no
  TYPE=eth
  CONFIG_WIRELESS=no
  BOOTPROTO=static
  SYSTEMD_BOOTPROTO=static
  CONFIG_IPV4=yes
  ```
  Указываем адрес в файле ipv4address

  ``` bash
  echo 192.168.200.10/28 > /etc/net/ifaces/<имя интерфейса>/ipv4address
  ```
  Указываем маршрут по умолчанию в ipv4route

  ``` bash
  echo default via 192.168.200.1 > /etc/net/ifaces/<имя интерфейса>/ipv4route
  ```

  Примечание:

  Для работоспособности необходимо выбрать один из двух сетевых менеджеров - systemd-networkd и network (etcnet). Systemd-networkd блокирует назначение адреса после    перезагрузки для network.

   Настройка файла для systemd-networkd
  ``` yml
  [Match]
  Name = enp0s3

  [Network]
  IPv6AcceptRA = false
  Address = 192.168.0.196/24
  Gateway = 192.168.0.1
  Domains = test.alt
  DNS = 8.8.8.8
  ```
  Оставляю ссылки на статьи:

  [Настройка сети](https://www.altlinux.org/%D0%9D%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B0_%D1%81%D0%B5%D1%82%D0%B8#Etcnet)

  [Systemd-networkd](https://www.altlinux.org/Systemd-networkd)
  




| Имя устройства | IP - Адрес  | Интерфейс | Шлюз по умолчанию | 
|:---------:|:--------------------------:|:--------------------------:|:--------------------:|
|     ISP     |           192.168.0.50/24         |   ens18   |  192.168.0.1        |        
|     ISP     |           172.16.4.1/28          |   ens19 |    172.16.4.2       |        
|     ISP     |           172.16.5.1/28           |   ens20  |   172.16.5.2         |        
|     HQ-CLI     |           172.16.200.10/28            |   ens18   |  172.16.200.1        |  
|     HQ-SRV  |           172.16.100.10/26            |   ens18    | 172.16.100.1       |        
|     BR-SRV     |           172.16.50.10/27          |    ens18   | 172.16.50.1       |  
|     HQ-RTR     |           172.16.50.10/27          |    ens18   | 172.16.50.1       |  
|     HQ-RTR     |           172.16.50.10/27          |    ens18   | 172.16.50.1       |  
|     HQ-RTR     |           172.16.50.10/27          |    ens18   | 172.16.50.1       |  
|     HQ-RTR     |           172.16.50.10/27          |    ens18   | 172.16.50.1       |  
|     BR-RTR    |           172.16.50.10/27          |    ens18   | 172.16.50.1       |  
|     BR-RTR    |           172.16.50.10/27          |    ens18   | 172.16.50.1       |  
