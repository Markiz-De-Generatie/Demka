

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

  




| Имя устройства | IP - Адрес  | Шлюз по умолчанию | 
|:---------:|:--------------------------:|:--------------------:|
|     ISP     |           HQ-SRV           |        r - -         |        
|     BR-SRV     |           r w -            |        r - -         |        
|     HQ-SRV     |           r w -            |        r - -         |        
|     HQ-CLI     |           r w -            |        r - -         |  
|     HQ-RTR    |           r w -            |        r - -         |        
|     BR-RTR     |           r w -            |        r - -         |  



