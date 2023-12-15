---
title: "Утечка исходного кода через директорию .git"
description: "Утечка исходного кода через директорию .git"
date: 2023-12-15T00:15:26+03:00
showTableOfContents: true
image: "https://www.0x29a.in/avatar.png"
draft: false
hideToc: false
tags: ["git", "hacking", "security"]
---

## Intro
Сегодня поговорим о раскрытии исходного кода через открытую директорию .git в веб-приложениях/рабочей среде.  
Хакеры могут использовать инструменты для загрузки/восстановления репозитория, чтобы получить доступ к исходному коду вашего приложения.

<!--more-->

## Почему это стоит искать
Это интересно потому что:
- Его легко обнаружить.
- Анализ исходного кода может выявить другие уязвимости, более критичные.
- Любой может использовать ваш исходный код в злонамеренных целях, нанеся вам финансовый/репутационный ущерб.
- Поиск файлов, содержащих конфиденциальную информацию, например учетные данные, токены, новые конечные точки и т. д.

Согласно [исследованию Truffle Security](https://trufflesecurity.com/blog/4500-of-the-top-1-million-websites-leaked-source-code-secrets/), 4,500 сайтов из миллионного топа раскрывают исходный код.

## Зачем нужен Git?
Git - это система контроля версий с открытым исходным кодом. Она широко используется разработчиками для отслеживания изменений, вносимых как в проекты с открытым исходным кодом, так и в коммерческие проекты. Разработчики могут использовать Git с основными операционными системами и средами разработки. 

## Что такое .git?
Папка .git содержит всю информацию, необходимую для контроля версий вашего проекта, а также все сведения о коммитах, адресе репозитория и прочем. В ней также находится лог, в котором хранится история коммитов. [(Подробнее о .git)](https://ru.hexlet.io/courses/intro_to_git/lessons/working-directory/theory_unit)

## Как найти
Если нет конкретной цели, то для поиска .git мы можем использовать Google Dorks:
```text
"index of” inurl:.git
```

```text
allintext:index filetype:git
```

Если мы уже ищем что-то целенаправленно, можем указать `site:`:
```text
"index of” inurl:.git site:example.com
```

```text
allintext:index filetype:git site:example.com
```

Так же для этой цели мы можем использовать [gobuster](https://github.com/OJ/gobuster)/[dirbuster](https://github.com/KajanM/DirBuster) (оба инструмента есть в дистрибутивах Kali/Parrot).

## Коды ошибок 
- 200 Found: отлично, работаем дальше. 
- 404 Not found: .git не существует на сервере или выбран неверный путь
- 403 Forbidden: .git существует, но на сервере выключен листинг.

## Эксплуатация
Если на  сервере включен листинг каталогов, мы можем использовать всего одну простую команду для загрузки всех файлов:  
```bash
wget --mirror -I .git example.com/.git/
```
После чего можно посмотреть изменения используя `git diff`.

Но листинга каталогов может и не быть, так что удобней для этого использовать инструменты:
- [git-dumper](https://github.com/arthaud/git-dumper)
- [GitTools](https://github.com/internetwache/GitTools)
- [GitHacker](https://github.com/WangYihang/GitHacker)

Если каталог `.git` открыт, он выглядит следующим образом:
![](../../static/source_code_leak_through_git1.png)

После завершения загрузки мы можем просмотреть статус всех локальных изменений и сравнить их с данными, полученными в репозитории целевого веб-сервера.
> Файлы отредактировны, имя хоста и репозитория скрыты.

![](../../static/source_code_leak_through_git2.png) ![](../../static/source_code_leak_through_git3.png)

Cтатус показывает только удаленные файлы, поскольку у нас есть только папка `.git`, загруженная с веб-сервера. Впрочем, это не проблема. Выполнение команды `git checkout -- .` или `git restore .` вернёт репозиторий к последнему изменению.

## Что делать дальше?
Торчащий наружу `.git` это сама по себе неприятная уязвимость, но чтобы не ходить два раза, можем немного пробежаться по репозиторию.
Что можно сделать с найденным репозиторием:
- Поискать CVE и уязвимости в коммитах используя [semgrep](https://github.com/semgrep/semgrep) или [git-vuln-finder](https://github.com/cve-search/git-vuln-finder)
- Поискать секреты используя [trufflehog](https://github.com/trufflesecurity/trufflehog) или [repo-security-scanner](https://github.com/UKHomeOffice/repo-security-scanner) 
- Составить поверхность атаки на приложение используя [noir](https://github.com/noir-cr/noir)
- Если используются контейнеры, можно использовать [trivy](https://github.com/aquasecurity/trivy)

## Как устранить уязвимость раскрытия исходного кода .git?
Чтобы устранить эту уязвимость, либо удалите папку git с вашего веб-сервера, либо запретите доступ к папкам `.git`.
Запретить доступ к папкам `.git` просто:

### Apache
Правим `.htaccess`.
```apache
<DirectoryMatch “^/.*/\.git/”> 
	Order deny, allow 
	Deny from all 
</DirectoryMatch>
```

### Nginx
Правим `nginx.conf`.
```nginx
location ~ /.git/ {
       deny all; 
}
```

### Lighttpd
Правим `lighttpd.conf`.
```lighttpd
server.modules += ( "mod_access" )

$HTTP["url"] =~ "^/\.git/" {
      url.access-deny = ("") 
}
```

## Outro
Как мы можем увидеть, обнаружение и устранение выглядят довольно просто. Однако проблемы, которые могут принести утекшие исходники представить довольно несложно.
- Если утёк сайт с платежами, то злоумышленники могут развернуть фишинговый ресурс;
- Утекшие секреты в коде могут обеспечить злоумыленникам доступ, в том числе и к внутренним\непубличнм сервисам;
- Утекшие секреты от API, которые в лучшем случае принесут финансовые потери.

---

## Reference
[Truffle Security Co: 4,500 of the Top 1 Million Websites Leaked Source Code, Secrets](https://trufflesecurity.com/blog/4500-of-the-top-1-million-websites-leaked-source-code-secrets/)  
[Введение в Git - Рабочая директория](https://ru.hexlet.io/courses/intro_to_git/lessons/working-directory/theory_unit)  
[How to Block .git Directory in Apache/Nginx](https://tecadmin.net/block-git-directory-in-apache-nginx/)  
