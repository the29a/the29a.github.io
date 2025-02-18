---
title: Таможня по-сомалийски. Вскрываем контейнеры Docker
subtitle:
date: 2025-02-08T16:48:43+03:00
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
  - docker
categories:
  - docker
  - vulnerabilities
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
  enable: false
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

## Intro
Docker-контейнеры давно и плотно вошли в нашу жизнь, их можно встретить как в кровавом Enterprise и разработке, так и на роутерах (например Keenetik и Mikrotik на ARM) и встроенных устройствах. Однако у удобства есть и обратная сторона: не совсем очевидно, что же содержат эти образы. Да и редко кто проверяет содержимое образа, а порой и вовсе используют неофициальные образы от непонятных авторов. 
Или может утрачены исходники и нужно пересоздать образ. Или просто любопытно, что же в образе содержится.

В любом случае, образ Docker это не чёрный ящик, и заглянуть под капот образа не большая проблема.

<!--more-->

## Тестовый образ
Для начала нам нужен образ, который будет выступать в роли подопытного. Базовый образ нам не подойдёт в силу малого количества слоёв. Мы будем использовать специально подготовленный тестовый образ.

Тестовый образ мы можем получить двумя путями:
- Спуллить мой готовый образ:
- Собрать самостоятельно, напишем Dockerfile сами добавим файлов, секретов, а после соберём и разберем его.

В роли наполнения образа будем использовать:
- небольшой бинарный файл
- небольшое веб-приложение с Python/Flask/Sqlite (взял [тут](https://github.com/rcormie/python-flask-sqlite-app-docker), но с исправлениями от DeepSeek)
- `.env` с тестовыми секретами
- секреты в `Dockerfile`

> Для желающих повторить все файлы будут в репозитории: [https://github.com/the29a/somali_customs](https://github.com/the29a/somali_customs)

### Подготовка тестового образа
#### Использование готового образа
Забираем образ с ghcr:
```shell
docker pull ghcr.io/the29a/somali_customs:latest
```
Если вы будете использовать готовый образ, то процесс сборки можно пропустить.

#### Сборка образа
Создадим бинарь:
```c
#include <stdio.h>

int main() {
	// Задаём строку, которую можно будет посмотреть strings
    static const char initial[] = "$3cr3t_$tring";
    // Чтобы был какой-то функционал, пусть выводит Hello, Cult of Wire
    printf("Hello, Cult of Wire\n");
    return 0;
}

```

Скомпилируем:
```shell
gcc findme.c -o findme
```

Добавим .env:
```shell
SECRET_1="578cb981-12b1-4931-9b74-9ff97b540b1e"
SECRET_2="c7784065-c83d-4275-8580-7f241f173d40"
```

Соберём всё в Dockerfile:
```dockerfile
FROM ubuntu:20.04
LABEL maintainer="the29a"

# Добавляем тестовые переменные
ENV USER="super_user"
ENV PASSWORD="super_secret_password"
ENV API_TOKEN="71738e41-a648-41c8-9bcf-755dcf970788"

# Устанавливаем переменные окружения для веб-приложения
ENV FLASK_ENV=development
ENV FLASK_APP=/src/app.py
ENV DATABASE=/src/database.db

ADD findme /usr/local/bin/findme
ADD .env .env

# Устанавливаем рабочую директорию
WORKDIR /src

# Копируем исходный код в контейнер
COPY ./src /src

# Устанавливаем необходимые пакеты
RUN apt-get update && \
    apt-get install -y \
    python3 \
    python3-pip \
    sqlite3 \
    curl && \
    pip3 install --upgrade pip==23.1.2 && \
    pip3 install -r /src/requirements.txt

# Открываем порт 5000
EXPOSE 5000

# Запускаем приложение с помощью Gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

И соберём наш образ:
```shell
docker build -t somali_customs .  
```

(Опционально) Можно запустить и потыкать:
```shell
docker run -p 5000:5000 somali_customs
```


## Общая информация и переменные окружения 
А теперь когда у нас есть образ, представим, что его собирали не мы (если брали готовый образ, разумеется). Мы о нём ничего не знаем, нашли его в приватном или публичном registry и нам нужно извлечь какие-то данные или пересоздать образ. 

Для начала, мы можем посмотреть информацию о нашем образе:
```shell
docker inspect somali_customs
```
И в ответ мы получим множество информации, вроде тэга, даты создания, а так же переменные окружения, которые были указаны в Dockerfile.

Либо мы сразу можем посмотреть, что указано в env:
```shell
# Так
docker inspect somali_customs --format '{{.Config.Env}}'

# Или так
docker inspect somali_customs | jq -r '.[].Config.Env[]'
```

В ответ получим переменные окружения, в том числе и секреты, указанные в Dockerfile.
```shell
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
USER=super_user
PASSWORD=super_secret_password
API_TOKEN=71738e41-a648-41c8-9bcf-755dcf970788
FLASK_ENV=development
FLASK_APP=/src/app.py
DATABASE=/src/database.db
```
> По этой причине и не рекомендуется передавать секреты в Dockerfile.

Но секретов из env недостаточно, или их может не быть вовсе.
Так что мы идём дальше.
![[Pasted image 20250212181500.png]]


## История слоёв
Для того, чтобы восстановить порядок действий, мы можем посмотреть историю слоёв. В history попадает не всё, например `FROM` отображаться не будет. Это связано с тем, что `FROM` - это инструкция Dockerfile, которая указывает базовый образ, но она не создает отдельный слой в финальном образе. Вместо этого история слоев начинается с первого слоя, который был добавлен поверх базового образа.

Для просмотра history у нас есть несколько вариантов:

### docker history
Для начала мы можем посмотреть историю с помощью `docker history`:
```shell
docker history somali_customs

IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
6dead3c50500   2 hours ago    CMD ["gunicorn" "--bind" "0.0.0.0:5000" "--w…   0B        buildkit.dockerfile.v0
<missing>      2 hours ago    EXPOSE map[5000/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      2 hours ago    RUN /bin/sh -c apt-get update &&     apt-get…   417MB     buildkit.dockerfile.v0
<missing>      2 hours ago    COPY ./src /src # buildkit                      17kB      buildkit.dockerfile.v0
<missing>      2 hours ago    WORKDIR /src                                    0B        buildkit.dockerfile.v0
<missing>      2 hours ago    ADD .env .env # buildkit                        96B       buildkit.dockerfile.v0
<missing>      2 hours ago    ADD findme /usr/local/bin/findme # buildkit     16kB      buildkit.dockerfile.v0
<missing>      2 hours ago    ENV DATABASE=/src/database.db                   0B        buildkit.dockerfile.v0
<missing>      2 hours ago    ENV FLASK_APP=/src/app.py                       0B        buildkit.dockerfile.v0
<missing>      2 hours ago    ENV FLASK_ENV=development                       0B        buildkit.dockerfile.v0
<missing>      2 hours ago    ENV API_TOKEN=71738e41-a648-41c8-9bcf-755dcf…   0B        buildkit.dockerfile.v0
<missing>      2 hours ago    ENV PASSWORD=super_secret_password              0B        buildkit.dockerfile.v0
<missing>      2 hours ago    ENV USER=super_user                             0B        buildkit.dockerfile.v0
<missing>      2 hours ago    LABEL maintainer=the29a                         0B        buildkit.dockerfile.v0
<missing>      4 months ago   /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B        
<missing>      4 months ago   /bin/sh -c #(nop) ADD file:7486147a645d8835a…   72.8MB    
<missing>      4 months ago   /bin/sh -c #(nop)  LABEL org.opencontainers.…   0B        
<missing>      4 months ago   /bin/sh -c #(nop)  LABEL org.opencontainers.…   0B        
<missing>      4 months ago   /bin/sh -c #(nop)  ARG LAUNCHPAD_BUILD_ARCH     0B        
<missing>      4 months ago   /bin/sh -c #(nop)  ARG RELEASE                  0B  
```

Данных не очень много, они идут снизу вверх. В целом малопригодно и требуется обработка. 
Так же вы могли обратить внимание на `<missing>`, с 2016 года это ожидаемое поведение.
Если интересно, то стоит почитать статью Нигеля Брауна [Explaining Docker Image IDs](https://windsock.io/explaining-docker-image-ids/) (или в частичном переводе [тут](https://questu.ru/questions/35310212/))
Краткая цитата:
> Я думаю, это ожидаемо; хранилище с адресацией к содержимому больше не использует «родительские» изображения для объединения слоев изображений.

Но эту проблему мы сможем решить с помощью dive, о котором чуть позже.

### dedockify
[Dedockify](https://github.com/mrhavens/Dedockify/) - небольшой скрипт, позволяющий получить список слоёв из history через Docker API. Можно использовать как из контейнера, так и сам py-скрипт:
```shell
git clone https://github.com/mrhavens/Dedockify.git 
# для запуска требуется передавать image_id, а не имя/тэг
python3 dedockify.py 6dead3c50500 
```

На выходе получаем обработанный список действий, из которого уже можно собирать Dockerfile. 
```dockerfile
FROM ghcr.io/the29a/somali_customs:latest
ARG RELEASE
ARG LAUNCHPAD_BUILD_ARCH
LABEL org.opencontainers.image.ref.name=ubuntu
LABEL org.opencontainers.image.version=20.04
ADD file:7486147a645d8835a5181c79f00a3606c6b714c83bcbfcd8862221eb14690f9e in /
CMD ["/bin/bash"]
RUN LABEL maintainer=the29a
RUN ENV USER=super_user
RUN ENV PASSWORD=super_secret_password
RUN ENV API_TOKEN=71738e41-a648-41c8-9bcf-755dcf970788
RUN ENV FLASK_ENV=development
RUN ENV FLASK_APP=/src/app.py
RUN ENV DATABASE=/src/database.db
RUN ADD findme /usr/local/bin/findme # buildkit
RUN ADD .env .env # buildkit
RUN WORKDIR /src
RUN COPY ./src /src # buildkit
RUN RUN /bin/sh -c apt-get update \
    &&     apt-get install -y     python3     python3-pip     sqlite3     curl \
    &&     pip3 install --upgrade pip==23.1.2 \
    &&     pip3 install -r /src/requirements.txt # buildkit
RUN EXPOSE map[5000/tcp:{}]
RUN CMD ["gunicorn" "--bind" "0.0.0.0:5000" "--workers" "4" "app:app"]
```


### zarva
[zarva](https://github.com/the29a/zarva) - мой небольшой скрипт, работающий через Docker API. Работает из Docker-контейнера и самостоятельным скриптом. 
```shell
git clone https://github.com/the29a/zarva.git && cd zarva
python3 zarva.py -lh somali_customs
```

Выхлоп выглядит чуть похуже, но в целом тоже пригодно для работы.
```dockerfile
ARG RELEASE
ARG LAUNCHPAD_BUILD_ARCH
LABEL org.opencontainers.image.ref.name=ubuntu
LABEL org.opencontainers.image.version=20.04
ADD file:7486147a645d8835a5181c79f00a3606c6b714c83bcbfcd8862221eb14690f9e in /
CMD ["/bin/bash"]
LABEL maintainer=the29a
ENV USER=super_user
ENV PASSWORD=super_secret_password
ENV API_TOKEN=71738e41-a648-41c8-9bcf-755dcf970788
ENV FLASK_ENV=development
ENV FLASK_APP=/src/app.py
ENV DATABASE=/src/database.db
ADD findme /usr/local/bin/findme # buildkit
ADD .env .env # buildkit
WORKDIR /src
COPY ./src /src # buildkit
RUN /bin/sh -c apt-get update &&     apt-get install -y     python3     python3-pip     sqlite3     curl &&     pip3 install --upgrade pip==23.1.2 &&     pip3 install -r /src/requirements.txt # buildkit
EXPOSE map[5000/tcp:{}]
CMD ["gunicorn" "--bind" "0.0.0.0:5000" "--workers" "4" "app:app"]
```


### dive
[dive](https://github.com/wagoodman/dive) - инструмент для анализа образов Docker. Готовый Dockerfile он нам не выдаст, но он решит проблему `<missing>` дайджеста образа. 
```shell
dive somali_customs:latest
```

![](../../static/Screenshot_2025-02-13_01-21-30.png)

## Извлекаем файлы
### Подготовка к вскрытию
Для извлечения файлов у нас есть два пути:
- извлечение файлов из образа, предварительно сохранив его как архив
- копирование из запущенного контейнера 

Для того, чтобы добраться до файлов в образе, нам будет достаточно встроенных средств. Для этого нужно сохранить образ как tar-архив и распаковать его.

```shell
# Если собирали образ самостоятельно
docker save somali_customs -o somali_customs.tar

# Для образа с ghcr
docker save ghcr.io/the29a/somali_customs  -o somali_customs.tar

mkdir -p somali_customs
tar xfv somali_customs.tar -C somali_customs
```

И после распаковки нас ждёт следующая картина:
```shell
ls -lah
total 28K
drwxrwxr-x 3 the29a the29a 4.0K Feb 12 23:33 .
drwxrwxr-x 4 the29a the29a 4.0K Feb 12 23:33 ..
drwxr-xr-x 3 the29a the29a 4.0K Feb 12 20:41 blobs
-rw-r--r-- 1 the29a the29a  370 Feb 12 23:32 index.json
-rw-r--r-- 1 the29a the29a 2.0K Jan  1  1970 manifest.json
-rw-r--r-- 1 the29a the29a   31 Jan  1  1970 oci-layout
-rw-r--r-- 1 the29a the29a   96 Jan  1  1970 repositories
```

### Что внутри
И что же у нас под капотом?
- `blobs` содержит все данные, связанные с образом, упакованные в виде объектов. Эти объекты хранятся в поддиректориях, названных по их хэшу 
- `index.json` является точкой входа для описания образа. Он содержит ссылки на манифесты (файлы `manifest.json`), которые, в свою очередь, описывают структуру образа.
- `manifest.json` описывает структуру конкретного образа. Он связывает слои (из `blobs`) с конфигурацией образа
- `oci-layout` указывает, что данная директория соответствует спецификации OCI. Он содержит минимальную информацию о версии спецификации
- `repositories` содержит информацию о тэгах образов. Он связывает имена образов и тэгов с соответствующими манифестами

Нас интересует blobs. blobs содержит три основных типа объектов:
- Слои: файлы, представляющие изменения файловой системы (добавление файлов, установка пакетов)
- Конфигурации: JSON-файлы, описывающие метаданные образа (например, команды `CMD`, переменные окружения `ENV`, рабочая директория `WORKDIR` и т.д.).
- Манифесты: JSON-файлы, описывающие структуру образа и связывающие слои с конфигурациями.

Содержимое blobs выглядит примерно так:
```shell
-rw-r--r-- 1 the29a the29a 424665600 Feb 12 20:41 21784e90071f6def6f38213e1bb1b5aaf59334e9e7044cb56d726d8b65dbe9f9
-rw-r--r-- 1 the29a the29a      1246 Feb 12 20:41 2c35ff2f5c0f1f315b5200a967ce0c5e5f30267e15497eda6ffe34de9285c6db
-rw-r--r-- 1 the29a the29a       482 Feb 12 20:41 53e6285237a453ff289810309a0658b8bb2daaa95f57195cd2841e4a7dd3d7c8
-rw-r--r-- 1 the29a the29a       482 Feb 12 20:41 5c0360e4bb03e58b69958f55afcd7a248d3993d23d25aa7043e4f65200a5b32d
-rw-r--r-- 1 the29a the29a      4122 Feb 12 20:41 6dead3c505007b4d2d80fc6cc19d3f8cc28650784a93477421936f17eb401d57
-rw-r--r-- 1 the29a the29a       482 Feb 12 20:41 7a2aea30127354ab1b44ed67a62f92d6c8cc71b46ef26c7f752fa5b3612df25c
-rw-r--r-- 1 the29a the29a       406 Feb 12 20:41 8c746c362d60c315f77e6cadeeb65a4ef38bf7d2ba3746f0ee7bed227ccf8a26
-rw-r--r-- 1 the29a the29a     23040 Feb 12 20:41 918282f295a57899a4c3071457fc62d15c962e76f9a84cb7247dbfc3f81228e0
-rw-r--r-- 1 the29a the29a     19456 Feb 12 20:41 adcede8a99681fb55938bc0aa1bc69704300c9c17efc129cd4f04fb66c8ebefa
-rw-r--r-- 1 the29a the29a      1159 Jan  1  1970 af187335483809b3e75feb10365c504c9120df31c89d59e7232fb752cd34f68f
-rw-r--r-- 1 the29a the29a      1536 Feb 12 20:41 e1803fc36b88d0fa86a30df1998e1a8db2c354c02a3a6830d04bb6a5f4317111
-rw-r--r-- 1 the29a the29a       482 Feb 12 20:41 e24771e2ed761daa950df0097e84756bd7bf17d238b7d665aef0879f7c51217e
-rw-r--r-- 1 the29a the29a      2048 Feb 12 20:41 f1080c8ed328be869717458a91a9bb4b0562f2cb31eea0beeee841af0e2cc56f
-rw-r--r-- 1 the29a the29a  75190784 Feb 12 20:41 fffe76c64ef2dee2d80a8bb3ad13d65d596d04a45510b1956a976a69215dae92
```

Чтобы было немного проще, можем задать расширение файлам небольшим скриптом:
```shell
#!/bin/bash

for file in *; do
    if [[ "$file" != *.json && "$file" != *.tar ]]; then
        if file "$file" | grep -q "JSON text data"; then
            mv "$file" "$file.json"
        elif file "$file" | grep -q "tar archive"; then
            mv "$file" "$file.tar"
        fi
    fi
done
```

Что нам делать дальше? А дальше всё довольно просто: мы знаем Digest слоя (если не указан в history, то мы можем посмотреть в dive), а дальше распаковываем как обычный tar.
Например, слой с командой `ADD .env .env` имеет Digest `f1080c8ed328be869717458a91a9bb4b0562f2cb31eea0beeee841af0e2cc56f`

Извлекаем файл и смотрим наш .env:
```shell
tar xfv f1080c8ed328be869717458a91a9bb4b0562f2cb31eea0beeee841af0e2cc56f.tar 

cat .env                     
SECRET_1="578cb981-12b1-4931-9b74-9ff97b540b1e"
SECRET_2="c7784065-c83d-4275-8580-7f241f173d40"
```

Так же находим наш findme в `6136ad49dff9e25bb042db7fbc9efecdd182620d0cb9af0503b7018638635df3`
```shell
tar xfv 6136ad49dff9e25bb042db7fbc9efecdd182620d0cb9af0503b7018638635df3.tar 

strings usr/local/bin/findme | sed '13,14!d'
Hello, Cult of Wire
$3cr3t_$tring
```

### Копирование из контейнера
Запускаем контейнер:
```shell
docker run -td ghcr.io/the29a/somali_customs                                    
f31355d24c80bac47d661c53db38adc17aa594df850864812fb37e1c7be9c6c9
```

Проверяем, что контейнер  не упал и работает:
```shell
docker ps -a 
CONTAINER ID   IMAGE                           COMMAND                  CREATED          STATUS          PORTS      NAMES
f31355d24c80   ghcr.io/the29a/somali_customs   "gunicorn --bind 0.0…"   12 minutes ago   Up 12 minutes   5000/tcp   magical_ritchie
```

Имея под рукой dive и\или зная, какие файлы у нас есть, мы просто копируем их из контейнера на хостовую машину: 
```shell
docker cp f31355d24c80:.env .
Successfully copied 2.05kB to /home/the29a/Dev/somali_customs/.
```

Проверяем полученный `.env`:
```shell
cat .env        
SECRET_1="578cb981-12b1-4931-9b74-9ff97b540b1e"
SECRET_2="c7784065-c83d-4275-8580-7f241f173d40"
```

Так же поступаем с нашим бинарником:
```shell
docker cp f31355d24c80:/usr/local/bin/findme .    
Successfully copied 17.9kB to /home/the29a/test/.
```

Проверяем содержимое:
```shell
strings findme | sed '13,14!d' 
Hello, Cult of Wire
$3cr3t_$tring
```

## Outro
Как мы смогли убедиться, в образах Docker нет магии, а в извлечении полезной для нас информации особой сложности нет. Да, в силу особенностей это немного не очевидно, но вполне решаемо. Упомянутые в статье инструменты хоть и не дают готовое решение из коробки, но сильно помогают в нашей задаче.

## Ссылки
### Инструменты
[https://github.com/mrhavens/Dedockify/](https://github.com/mrhavens/Dedockify/)  
[https://github.com/wagoodman/dive](https://github.com/wagoodman/dive)  
[https://github.com/the29a/zarva](https://github.com/the29a/zarva)  
[https://github.com/containers/skopeo](https://github.com/containers/skopeo)  
[Docker SDK for Python](https://docker-py.readthedocs.io/en/stable/)  

### Статья на схожую тему
[Gcore: Reverse Engineer Docker Images into Dockerfiles](https://gcore.com/learning/reverse-engineer-docker-images-into-dockerfiles-with-dedockify/)  

### Прочее
[Explaining Docker Image IDs](https://windsock.io/explaining-docker-image-ids/)   
[https://github.com/rcormie/python-flask-sqlite-app-docker](https://github.com/rcormie/python-flask-sqlite-app-docker)  