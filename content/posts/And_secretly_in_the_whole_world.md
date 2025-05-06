---
title:  По секрету всему свету или всё, что случается в git остаётся в git (не совсем)
subtitle:
date: 2025-05-06T18:26:25+03:00
slug: 83dfd55
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
  - security
categories:
  - security
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
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
## Intro
Представим себе ситуацию: мы что-то в спешке коммитим в git и по какой-то причине в наш коммит попадает пароль, файл с токенами или какой-то иной секрет. И кажется, что логичное решение это удалить последний коммит.

<!--more-->

```sh
git reset --hard HEAD~1
git push origin main --force
```

Но `git push --force` может быть запрещен (как минимум для ветки main), да и в целом это не совсем верный подход, даже если вы работаете над своим небольшим проектом в гордом одиночестве.  И это в целом не про удаление секретов, а про отмену последнего коммита и его применение в удаленном репозитории, через `git reflog` доступ к коммитам остаётся. 

Например:
```shell
git reflog
git reset --hard your_hash_id
git branch new_branch_name your_hash_id
git log -p -2
```

**Захардкоженый секрет == скомпроментированный секрет**, так что его не только стоит удалить (тут скорей больше для сканеров, чтобы не фолзили), но и менять секрет.


## Где искать эти ваши секреты?
Где в git можно найти секреты:
- в CI/CD
- в самом коде и конфигах
Это мы можем покрыть обычными сканерами секретов, так у нас всё как код.

На каких этапах взаимодействия с git мы можем обнаружить захардкоженные секреты:
- При коммите (pre-commit)
- При пуше (pre-push)
- При получении изменений (pre-receive)

Из хуков пожалуй будет интересен pre-commit, его и рассмотрим. Но для любопытных есть GitGuardian есть документация по использованию [ggshield](https://github.com/GitGuardian/ggshield) в [pre-push](https://docs.gitguardian.com/ggshield-docs/integrations/git-hooks/pre-push) и [pre-receive](https://docs.gitguardian.com/ggshield-docs/integrations/git-hooks/pre-receive).

## Pre-commit hook
Наиболее удобный вариант, но применим либо к разработчикам-одиночкам, либо к зрелым командам.
Для этого можно использовать различные инструменты, но из-за простоты возьму для примера Trufflehog:
```shell
# Создаём pre-hook
mkdir -p ~/.git-hooks
touch ~/.git-hooks/pre-commit
chmod +x ~/.git-hooks/pre-commit

# Добавляем следующее содержимое в ~/.git-hooks/pre-commit:

#!/bin/sh
trufflehog git file://. --since-commit HEAD --branch=main --results=verified,unknown --fail

# Добавляем хук в конфиг
git config --global core.hooksPath ~/.git-hooks
```
Теперь при коммите Trufflehog будет проверять на наличие секретов. 

> Это не является единственным решением или точкой проверки, так как при необходимости проверку можно обойти.

Так же для pre-commit хуков можно использовать фреймворк [pre-commit](https://pre-commit.com/) или [Husky](https://typicode.github.io/husky/).  
Trufflehog так же не лучшее решение, но об этом чуть ниже.

## Поиск секретов в кодовой базе
Pre-commit hook это отлично, но не гарантирует полное отсутствие секретов в коде. Для этого есть множество как OSS так и платных инструментов. Что-то можно так же использовать в Pre-commit hook, что-то в CI/CD, а что-то как sidecar-сервис.

Сами инструменты в целом похожи могут сканировать как локально (код, репозитории, конфиги), так и удалённые репозитории.
### Инструменты
- [Gitleaks](https://github.com/gitleaks/gitleaks) - Лёгкий и быстрый сканер для обнаружения токенов в Git-репозиториях. Идеально подходит для интеграции в CI/CD.
- [ggshield](https://github.com/GitGuardian/ggshield)  CLI-инструмент, которое запускается в локальной среде или в среде CI, обнаруживает более 400+ типов секретов. Но привязан к GitGuardian.
- [Detect Secrets](https://github.com/Yelp/detect-secrets) - Инструмент от Yelp с гибкой настройкой и системой плагинов. 
- [Talisman](https://github.com/thoughtworks/talisman) - Инструмент от ThoughtWorks, предотвращающий случайную отправку данных в репозиторий.
- [Secretlint](https://github.com/secretlint/secretlint) - Линтер секретов для `.env`, JSON, YAML и др. без зависимости от Git. Лучше запускать в Docker.
 - [TruffleHog](https://github.com/trufflesecurity/trufflehog) - Мощный инструмент для поиска, классификации и анализа утечек ключей и токенов в коде. Поддерживает более 800 типов секретов и может проверять их актуальность.
> [!WARNING]
> Сейчас OSS-версия Trufflehog имеет значительные ограничения по источникам сканирования. Из 21 источника в OSS поддерживается только 9. Подробнеей смотрите в [документации](https://docs.trufflesecurity.com/scan-data-for-secrets).

| Название                                                    | git | CI/CD | Local |
| ----------------------------------------------------------- | --- | ----- | ----- |
| [TruffleHog](https://github.com/trufflesecurity/trufflehog) | ✅  | ✅    | ✅    |
| [Gitleaks](https://github.com/gitleaks/gitleaks)            | ✅  | ✅    | ❌    |
| [ggshield](https://github.com/GitGuardian/ggshield)         | ✅  | ✅    | ✅    |
| [Detect Secrets](https://github.com/Yelp/detect-secrets)    | ✅  | ✅    | ❌    |
| [Talisman](https://github.com/thoughtworks/talisman)        | ✅  | ✅    | ✅    |
| [Secretlint](https://github.com/secretlint/secretlint)      | ✅  | ✅    | ✅    |

### Сервисы
- [GitGuardian](https://www.gitguardian.com) -  обнаруживает более 350 типов секретов как в публичных, так и в приватных репозиториях. Предлагает подробные отчёты и интеграции с различными платформами. Платный, но есть демо на 30 дней.
- [TruffleHog Enterprise](https://trufflesecurity.com/trufflehog-enterprise) - 20+ интеграций, 800+ типов секретов, демо нет. 

### Рекомендации
- Интеграция в CI/CD: `Gitleaks`, `ggshield`, `detect-secrets`.
- Pre-commit хуки: `detect-secrets`, `talisman`, `gitleaks`.
- Сканирование файлов вне Git: `detect-secrets`, `secretlint`, `ggshield`.  
Из-за ограничений сейчас Trufflehog рекомендовать не буду. Но на общем фоне вполне интересно выглядит `detect-secrets`, хоть он и требует особого подхода, но обновлялся он почти год назад. Так же неплохо работает `ggshield`, если вас не смущает привязка к GitGuardian.


## Удаляем неудаляемое
Секреты в коде найдены, заменены, но что делать, если их все в коде надо удалить? Удалять коммиты решение плохое, а если секретов больше чем 1, то задача кажется не выполнимой. Но и для этого уже есть решения.  
- [git filter-branch](https://git-scm.com/docs/git-filter-branch) - встроенный в git инструмент для изменения истории. Требует хорошего знания git и работает довольно медленно.  
- [git-filter-repo](https://github.com/newren/git-filter-repo) - инструмент для удаления секретов в коде.  
- [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) -  более простая и быстрая альтернатива git-filter-branch для очистки истории Git-репозитория.  

Возьмем для примера BFG Repo-Cleaner. Для начала нам нужен список секретов 
```sh
# Сканируем и генерируем отчёт в json
trufflehog git file://. --branch=main --results=verified,unverified,unknown -j > report.json

# Уберем лишнее, вроде ssh-ключей: 
cat report.json  | jq -r '.Raw | select (test("^-") |not)' > secrets.list

# Скачиваем bfg
curl https://repo1.maven.org/maven2/com/madgag/bfg/1.15.0/bfg-1.15.0.jar -o bfg.jar
```

Далее нам потребуется [bare](https://git-scm.com/docs/gitglossary.html#def_bare_repository) repo,что означает, что ваши обычные файлы не будут видны, но это полная копия базы данных Git вашего репозитория. И на данном этапе стоит сделать бэкап репозитория.

Так же для простоты использования `bfg` используется алиас для `java -jar bfg.jar` (например: `alias bfg='java -jar ~/.local/bin/bfg.jar'`)
```sh
# Клонируем репозиторий с опцией --mirror
git clone --mirror git://example.com/repo-with-secrets.git

# Удаляем ssh-ключи (если они были)
bfg --delete-files id_{dsa,rsa}  my-repo.git

# Удаляем секреты в репозитории, которые содержатся в подготовленном файле
bfg --replace-text secrets.list . repo-with-secrets.git

# Для защищенных веток (например main) добавляем флаг --no-blob-protection
bfg --replace-text secrets.list repo-with-secrets.git --no-blob-protection
```

BFG изменит ваши коммиты и все ветки и теги.  
После чего проведём небольшую уборку с `git reflog` и `git gc`.
```sh
cd some-big-repo.git
git reflog expire --expire=now --all && git gc --prune=now --aggressive
```

Если всё в порядке, можем запушить изменения (обратите внимание, что поскольку мы использовали `--mirror`флаг, этот push обновит **все** ссылки в репозитории) :
```shell
git push
```

## Outro
Как можно заметить, удаление коммита не является решением проблемы, а удаление секрета это недостаточная мера, его в любом случае необходимо ротировать.  
Покрыть только инструментами поиск секретов довольно сложно. Да и с увеличением количества репозиториев задача становится сложнее. Нельзя всё решить хуками или проверками в CI/CD.  Вариант с живущим параллельным сканером в целом неплоха, но лучше применять всё в комплексе.  
Секреты в принципе удаляемы, но всё же есть риск что-нибудь сломать.  

Не храните секреты в коде и конфигах (есть же встроенные средства Gitlab и Github, или даже Vault) и будет вам счастье.  

---
[Github Docs: Removing sensitive data from a repository](https://docs.github.com/ru/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)  
[Git-SCM: Customizing Git - Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)  
[TruffleHog Docs: Pre-commit hooks](https://docs.trufflesecurity.com/pre-commit-hooks)  
[Git-SCM: git-reflog](https://git-scm.com/docs/git-reflog)  
[Habr: Коммитить нельзя сканировать: как мы боремся с секретами в коде](https://habr.com/ru/companies/vk/articles/858022/)  