---
title: Карликовый сад для роботов. Ищем низко висящие фрукты в Android
subtitle:
date: 2024-02-12T11:23:13+03:00
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
  - android
  - security
  - hacking
  - jadx
  - apktool
categories:
  - android
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

Внутренний мир Android-приложений часто интересует баг-хантеров, пентестеров и прочих безопасников. Но не каждый понимает, как подступиться к приложению, на русском языке материала не так много, а на английском далеко не всё бывает в актуальном состоянии. Так что хотелось пройти этот путь самому, заодно поделиться опытом, который может кому-то помочь.

<!--more-->

> Я не являюсь профессиональным реверсером или Android разработчиком. Но хотел понять, что можно получить из Android-приложения не имея специальных навыков. 

Как выглядит процесс анализа: 
 - Получение apk-файла. Вариантов множество. Можно с телефона, скачать из F-Droid, воспользоваться утилитой вроде apkeep или apkpull .
 - Декомпилируем apk. Можно использовать apktool или jadx. Декомпилированный код можно выгрузить в виде проекта для Android Studio или иной IDE и в ней же открыть.
 - Анализируем исходники на предмет интересных конфигов (папка assets), строк (res/values/*) и всего, за что зацепится глаз.
 - ?????
 - Profit

## Получение apk-файла
В первую очередь нам нужно как-то получить сам apk-файл, для этого нам нужно знать [название пакета приложения](https://support.google.com/admob/answer/9972781?hl=ru), например мы можем его узнать из url магазина приложений, например Google Play.

Пример ссылки:
```url
https://play.google.com/store/apps/details?id=com.example.app
```

Дальше у нас есть два пути: получить apk с телефона, или скачать и магазина приложений.
С первым вариантом могут быть сложности, так как может потребоваться root.
### Получаем приложение с телефона по adb
Запрашиваем список установленных приложений:
```shell
adb shell pm list packages
# Можно использовать grep для удобства
adb shell pm list packages | grep example
package:com.example.app
```
Получаем полный путь до желаемого apk
```shell
adb shell pm path com.example.app
# Должно получиться что-то вроде такого
package:/data/app/~~fD3m3SCkscvlQCc3IpncKw==/com.example.app-K7_gdHCjcFTAyLk94z3Ttg==/base.apk
package:/data/app/~~fD3m3SCkscvlQCc3IpncKw==/com.example.app-K7_gdHCjcFTAyLk94z3Ttg==/split_config.arm64_v8a.apk
package:/data/app/~~fD3m3SCkscvlQCc3IpncKw==/com.example.app-K7_gdHCjcFTAyLk94z3Ttg==/split_config.ru.apk
package:/data/app/~~fD3m3SCkscvlQCc3IpncKw==/com.example.app-K7_gdHCjcFTAyLk94z3Ttg==/split_config.xxhdpi.apk
```

Скачиваем нужный apk-файл:
```shell
adb pull /data/app/~~fD3m3SCkscvlQCc3IpncKw==/com.example.app-K7_gdHCjcFTAyLk94z3Ttg==/base.apk
```

Так же можно использовать и готовые инструменты вроде [APKpull](https://github.com/david-lev/apkpull), который работает поверх adb.  
Для удобства можно использовать однострочник:
```shell
curl -sL bit.ly/apkpull | bash -s -- com.example.app
```

### Скачиваем apk из магазина приложений
Этот пункт сильно проще, мы его можем скачать руками из:
- [F-Droid](https://f-droid.org) (если оно там есть)
- [APKsFULL](https://apksfull.com/)
- [APKPure](https://apkpure.net/)

Можно скачать используя готовые инструменты:
- [apk-downloader](https://github.com/EngineerDanny/apk-downloader/) (использует APKsFULL)
- [apkeep](https://github.com/EFForg/apkeep) (скачивает с Play Store,  APKPure, F-Droid, Huawei AppGallery)

## Распаковка apk
Вне зависимости от цели анализа приложения, без этапа декомпиляции или распаковки не обойтись. В принципе, мы можем распаковать его как обычный архив (`unzip example.apk -d extract_dir`), но так ничего интересного мы не получим, и нам потребуются декомпиляторы. 

Инструментов может быть много, но мы будем использовать два основных:
- [apktool](https://github.com/iBotPeaches/Apktool)
- [jadx](https://github.com/skylot/jadx)

### [apktool](https://github.com/iBotPeaches/Apktool)
```shell
apktool d example.apk -o examlpe_apk_dir
```

### [jadx](https://github.com/skylot/jadx)
```shell
# По какой-то причине jadx не понимает относительные пути, только абсолютные, возможно проблема только у меня
jadx -d /home/the29a/examlpe_apk_dir /home/the29a/example.apk
```

Так же можно сразу открыть apk с помощью jadx-gui и сохранить как проект Gradle:
```
File -> Save as gradle project
```

### Что лучше: jadx vs. apktool
Вопрос хороший. Иметь лучше оба инструмента и уметь ими пользоваться. И желательно знать их особенности.
#### apktool
- Разбирает ресурсы до состояния, близкого к оригиналу и позволяет получить как можно больше ресурсов из приложения
- Apktool декомпилирует APK в Smali, что позволяет модифицировать и пересобрать apk
- Нет gui, быстрого просмотра структуры/файлов/текста.

#### jadx
- Позволяет увидеть структуру приложения после декомпиляции
- jadx может декомпилировать файлы .dex в файлы классов Java, обеспечивая читабельность кода и возможность проверки кода SAST 
- Возможность сохранения декомпилированного приложения в виде проекта Gradle
- Декомпиляция приложения может завершиться неудачей, если в приложении используются символы, отличные от ASCII


## Собираем низковисящие фрукты
В ходе анализа можно пройтись как обычными инструментами, там и специализированными. Например для анализа кода мы можем использовать какой-нибудь удобный для нас SAST (в моём случае это [semgrep](https://t.me/cultofwire/1179)), а так же разными инструментами, рассчитанными на работу с Android приложениями. 
Но для начала не будет лишним пробежаться самому, возможно в ходе ручного поиска найдется что-то интересное, что упустят инструменты.

### Ручной поиск
Автоматизированный поиск может пропустить некоторые интересные моменты, например секреты в `res/values/strings.xml`. Прочие xml-файлы в `res/values` тоже имеет смысл проверить самостоятельно.
Так же нам может быть интересен AndroidManifest.xml, файлы с расширением .sqlite и .db, содержимое директории assets.

### SAST
#### semgrep
[semgrep](https://github.com/semgrep/semgrep/) - довольно известный SAST-инструмент. Пройтись по коду каким-либо SAST лишним не будет. Не факт, что это будут простые или эксплуатируемые уязвимости, но находки могут быть интересными.
```shell
semgrep scan --config auto -o example_semgrep.log examlpe_apk_dir/
```

Есть отдельный репозиторий с правилами для Android - [github.com/mindedsecurity/semgrep-rules-android-security](https://github.com/mindedsecurity/semgrep-rules-android-security)
```bash
git clone https://github.com/mindedsecurity/semgrep-rules-android-security
jadx -d examlpe_apk_dir target.apk
cd semgrep-rules-android-security/
semgrep -c ./rules/ ..examlpe_apk_dir
```
> Правила работают не предсказуемо. На момент написания статьи правила могли вызывать ошибку:
> ```
> TypeError: unhashable type: 'list'
> ```
> jadx 1.4.7  
> semgrep 1.55.0 и1.59.0  


### Скрипты и инструменты
Начнем с обычных инструментов, переходя от простого к сложному.

#### apk2url
[apk2url](https://github.com/n0mi1k/apk2url) извлекает URL и IP конечных точек из APK-файла и выполняет фильтрацию в выходной файл .txt.  
Стоит учесть тот момент, что apk2url декомпилирует apk используя [jadx](https://github.com/skylot/jadx).
```shell
git clone https://github.com/n0mi1k/apk2url
./apk2url.sh /home/the29a/example.apk
```


#### apkleaks
[apkleaks](https://github.com/dwisiswant0/apkleaks) - cканирует APK на наличие URI/URL эндпоинтов и секретов. Инструмент давно не обновлялся, но всё ещё применим и актуален.  
apkleaks так же использует [jadx](https://github.com/skylot/jadx) для декомпиляции apk. 
 ```shell
# Можно установить с помощью pip
pip3 install apkleaks
# Или использовать в виде docker-контейнера
docker pull dwisiswant0/apkleaks:latest
```

```shell
apkleaks -f /home/the29a/example.apk
# Можно запустить сразу с репортом в файл
apkleaks -f /home/the29a/example.apk -o apkleaks.report 
```


### Фреймворки

#### APKHunt
[APKHunt](https://github.com/Cyber-Buddy/APKHunt) - это комплексный инструмент статического анализа кода приложений для Android, основанный на фреймворке [OWASP MASVS](https://mobile-security.gitbook.io/masvs/) v1.5.0. 
APKHunt охватывает большинство тестовых случаев SAST связанных с OWASP MASVS и обеспечивает низкий процент ложных срабатываний.

Сам распакует (jadx), сам проверит и выдаст подробный отчёт.
```shell
git clone https://github.com/Cyber-Buddy/APKHunt.git
cd APKHunt
go run apkhunt.go -p ~/example.apk
```
Но в силу особенностей, APKHunt работает только в Linux.

[Видео-демо работы](https://user-images.githubusercontent.com/32319538/211979260-194e858b-373a-4911-8c56-78d1d568d3aa.mp4).

#### MobFS
[MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF) - это автоматизированная система для тестирования мобильных приложений (Android/iOS/Windows), анализа вредоносного ПО и оценки безопасности, выполняющая статический и динамический анализ.

У MobFS есть удобный веб-интерфейс, что позволяет держать все результаты сканирования в одном месте, а так же работать с множеством приложений.
Так же при желании он интегрируется в CI/CD.

Разворачивается в docker:
```shell
docker pull opensecurity/mobile-security-framework-mobsf:latest
docker run -it --rm -p 8000:8000 opensecurity/mobile-security-framework-mobsf:latest
```

Документация: [mobsf.github.io/docs/](https://mobsf.github.io/docs/).  
Пощупать на тестовом приложении можно онлайн на [mobsf.live](https://mobsf.live/).

### Иные инструменты
Не мог обойти внимание инструменты которые хоть и будут бесполезны для новичков на начальных этапах, в дальнейшем могут стать полезными помощниками в работе.

#### mobileAudit
[MobileAudit](https://github.com/mpast/mobileAudit) - SAST для APK. MobileAudit фокусируется не только на тестировании безопасности и защитных сценариях и больше применим для анализа малвари:
- Статический анализ (SAST): Выполняет полную декомпиляцию APK и извлекает из него всю возможную информацию. Он сообщает о различных уязвимостях и находках в исходном коде, сгруппированных по различным категориям. Также имеется полная поддержка триажа находок.
- Анализ вредоносного ПО: находит опасные разрешения и подозрительный код.
- Best Practices of Secure Android Coding: подсказывает разработчикам, в каких частях кода они пишут безопасно, а в каких - нет.

[OWASP Mobile Audit](https://owasp.org/www-project-mobile-audit/)

#### Drozer
[Drozer](https://github.com/WithSecureLabs/drozer) -  система аудита безопасности и атак для Android.
Drozer позволяет получить информацию о приложении, запустить его активности, подключиться к [ContentProvider](https://developer.android.com/guide/topics/providers/content-providers), отправить сообщения сервису, предоставляя инструменты для использования публичных эксплойтов для Android.  
Всё, чтобы вытащить из приложения информацию или заставить его сделать то, что нам нужно через стандартные API и каналы коммуникации.  

Для взаимодействия drozer использует агент (который будет использоваться как RAT/reverse shell), устанавливаемый на телефон/эмулятор.  
Drozer давно не обновлялся, поэтому могут быть проблемы с установкой.   

[Withsecure: Drozer](https://labs.withsecure.com/tools/drozer)  
[Github: Drozer-Docker](https://github.com/Yogehi/Drozer-Docker)  
[hacktricks.xyz: Drozer Tutorial](https://book.hacktricks.xyz/mobile-pentesting/android-app-pentesting/drozer-tutorial)  
[Habr: Проверяем безопасность приложений с помощью Drozer](https://habr.com/ru/companies/alexhost/articles/535396/)  
  
## Тренируемся на уязвимых приложениях
Никто не запрещает работать сразу c приложениями из маркетов, но находить более сложные уязвимости вряд ли выйдет. Для этого есть специальные уязвимые приложения, написанные для образовательных целей.  
На чём потренироваться:  
[Github: DIVA Android - Damn Insecure and vulnerable App for Android](https://github.com/payatu/diva-android)  
[Github: Damn Vulnerable Bank](https://github.com/rewanthtammana/Damn-Vulnerable-Bank)  
[Github: Vuldroid - Vulnerable Android Application](https://github.com/jaiswalakshansh/Vuldroid)  

## Что почитать
### Материалы от OWASP
[OWASP Mobile Application Security](https://mas.owasp.org/)  
[OWASP Mobile Application Security Testing Guide](https://mobile-security.gitbook.io/mobile-security-testing-guide)  
### Статьи
[fi5t: Как вкатиться в безопасность android приложений в 2024](https://fi5t.xyz/posts/how-to-join-mobilesec-2024/)  
[hacktricks: Android APK Checklist](https://book.hacktricks.xyz/mobile-pentesting/android-checklist)  
### Telegram-каналы
[Mobile AppSec World](https://t.me/mobile_appsec_world)  
[Android Guards](https://t.me/android_guards_today)  
### Github-репозитории
[Github: awesome-android-security](https://github.com/saeidshirazi/awesome-android-security)  
[Github: Awesome Android Reverse Engineering](https://github.com/user1342/Awesome-Android-Reverse-Engineering)  

## Outro
Процесс реверса Android-приложений можно описать кратким "Easy To Learn But Hard To Master", так как без знания Java/Kotlin и знания разработки под Android сложно углубиться в поиск уязвимостей. Но для того, чтобы начать их искать нужно не так и много: желание, любопытство и пара инструментов.