---
title: Делаем отчёты nmap дружелюбней
subtitle:
date: 2023-12-24T17:00:40+03:00
draft: false
author:
  name:
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - nmap
categories:
  - nmap
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRss: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

Nmap инструмент мощный и гибкий, но его отчёты о сканировании далеко не всегда удобны для просмотра. Они не всегда наглядны, иногда их надо показывать людям, которые не готовы разглядывать вывод из stdin, а иногда и вовсе отчёт нужно конвертировать для дальнейшей обработки и\или хранения.  
Так же на nmap некоторые строят целые сканеры уязвимостей (например nerve) или инструменты для постройки майндмап, например [NMapify](https://t.me/cultofwire/1128), но это не всегда нужно, а порой и вовсе избыточно.
<!--more-->

Пойдем от простого к сложному:
- nmap2md
- nmap-formater
- vscode-nmap-peek
- WebMap

И в рамках истории:
- nerve

## nmap2md
[nmap2md](https://github.com/vdjagilev/nmap2md) - небольшая утилита для преобразования xml-отчётов nmap в markdown. Cо сложным репортом может не работать, но для простого сканирования, вроде инвентаризации своей сети вполне подойдет.

Проект заброшен, так что не стоит ожидать от nmap2md сложных вещей и исправлений багов.

Репозиторий: [https://github.com/vdjagilev/nmap2md](https://github.com/vdjagilev/nmap2md)

## nmap-formater
[nmap-formatter](https://github.com/vdjagilev/nmap-formatter) - инструмент, позволяющий конвертировать xml-отчёт сканирования nmap в html, csv, json, markdown, graphviz  или sqlite. Функционал довольно гибкий, так что стоит глянуть их Wiki. nmap-formater так же доступен как библиотека в golang. 

Репозиторий: [https://github.com/vdjagilev/nmap-formatter](https://github.com/vdjagilev/nmap-formatter)  
Wiki: [https://github.com/vdjagilev/nmap-formatter/wiki/Usage](https://github.com/vdjagilev/nmap-formatter/wiki/Usage)  
Use as a library: [https://github.com/vdjagilev/nmap-formatter/wiki/Use-as-a-library](https://github.com/vdjagilev/nmap-formatter/wiki/Use-as-a-library)  

## vscode-nmap-peek
Nmap-Peek - плагин для VSCode для просмотра xml-репортов Nmap. Плагин использует React, что (в теории) делает его менее тормозным. Кроме того, немного измененная версия плагина развернута на сайте автора, который отображает репорт в том же виде.

- Репозиторий: [https://github.com/marduc812/vscode-nmap-peek](https://github.com/marduc812/vscode-nmap-peek)  
- VScode marketplace: [https://marketplace.visualstudio.com/items?itemName=marduc812.nmap-peek](https://marketplace.visualstudio.com/items?itemName=marduc812.nmap-peek)  
- Online версия: [https://www.devoven.com/tools/nmap-viewer](https://www.devoven.com/tools/nmap-viewer)  

## nmap-bootstrap-xsl
[nmap-bootstrap-xsl](https://github.com/Haxxnet/nmap-bootstrap-xsl) - реализация Nmap XSL на Bootstrap. Позволяет преобразовывать результаты сканирования  Nmap XML в  HTML-отчеты.

Для конвертирования используется сам xls-файл и xsltproc:
```shell
# скачиваем  bootstrap xsl
wget https://raw.githubusercontent.com/Haxxnet/nmap-bootstrap-xsl/main/nmap-bootstrap.xsl

# конвертируем отчёт xml в html
xsltproc -o report.html nmap-bootstrap.xsl nmap.xml
```

Выглядит довольно громоздко, но при необходимости можно запускать сканирование с репортом в одну строку:
```shell
nmap -sS -Pn --stylesheet https://raw.githubusercontent.com/Haxxnet/nmap-bootstrap-xsl/main/nmap-bootstrap.xsl scanme.nmap.org
```

Репозиторий: [https://github.com/Haxxnet/nmap-bootstrap-xsl](https://github.com/Haxxnet/nmap-bootstrap-xsl)  
Демо-отчёт: [https://haxxnet.github.io/nmap-bootstrap-xsl/report.html](https://haxxnet.github.io/nmap-bootstrap-xsl/report.html)  

## WebMap
[WebMap](https://github.com/SabyasachiRana/WebMap) веб-дашборд для xml-репортов Nmap. 
Импортирует xml-отчёты и отрисовывает их в небольшой web-интерфейсе. Webmap удобно запускается из Docker, но можно собрать\запустить самому.

Функционал:
- Импорт и парсинг XML-файлов Nmap
- Запуск и планирование сканирования Nmap
- Статистика и графики по обнаруженным службам, портам, ОС
- Вставка заметок для конкретного узла
- Создание отчета в формате PDF с графиками, подробной информацией, метками и заметками.
- Копирование в буфер обмена команд для Nikto, Curl или Telnet.
- Поиск CVE и эксплойтов на основе CPE, собранных Nmap
- RESTful API
 
Проект давно не поддерживается, но вполне работоспособен и применим.

- Репозиторий: [https://github.com/SabyasachiRana/WebMap](https://github.com/SabyasachiRana/WebMap)

## nerve 
[NERVE Continuous Vulnerability Scanner](https://github.com/PaytmLabs/nerve)  
NERVE - сканер уязвимостей, предназначенный для поиска уязвимостей уровня "низковисящих фруктов" в  приложениях, сетевых службах и непропатченных сервисах.

NERVE предлагает следующие возможности:
- Web-интерфейс 
- REST API 
- Уведомления в Slack, на почту  и через вебхуки

Возможности NERVE по обнаружению уязвимостей:
- панели администрирования (Solr, Django, PHPMyAdmin)
- Открытые репозитории
- Раскрытие информации
- Заброшенные / дефолтные веб-страницы
- Неправильная конфигурация служб (Nginx, Apache, IIS и т. д.)
- SSH-серверы
- Открытые базы данных
- Индексированые каталоги

Последний коммит был три года назад, так что использование в текущее время большого смысла не имеет.

Не могу сказать, что мир потерял "феноменальный проект", но было бы приятно иметь альтернативы разным тяжеловесам и проприетарным сканерам.