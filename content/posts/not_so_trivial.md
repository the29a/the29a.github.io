---
title: Not so (trivy)al. Как использовать trivy на большом скоупе и не поехать кукухой (нет)
subtitle:
date: 2024-11-08T16:48:43+03:00
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
  - trivy
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

Стояла задача простая и ясная как летний день - сканировать Docker-образы в своём registry. 
Но по мере работы стали проявляться нюансы, которые как Газманов забирали этот ясный день:
- Тормозной и неудобный API Nexus (частная проблема, решается использованием Registry API);
- Большое количество и объем образов (3к+ образов и более 1.5Tb объема);
- Не все используют тэг latest;
- В роли сборщика репортов не подходил DefectDojo.

<!--more-->

## Intro
> [!info]
> Тут не будет готового решения, скорей концептуальное описание пройденного пути.
> Так же примеры кода будут в виде shell-скриптов, так как с языками программирования у меня не всё просто, а так же многое завязано за jq.
> В статье будет про Nexus в роли репозитория, но основные моменты могут быть применимы и к другим решениям.



## Выбор тулкита
Какие требования были к сканеру:
- Сканирование образов Registry;
- Сканирование Dockerfile в gitlab-репозитории и локально;
- Open-Source (Можно было бы всё купить, но обходимся тем, что есть);
- Отчёты в приемлемом виде (json, опционально html);
- Интеграция в CI/CD (скорей задел на будущее).

### Варианты выбора
Сканеры на выбор есть, но нельзя сказать, что их много.
#### grype
[grype](https://github.com/anchore/grype) - ранее известный как [Anchore-Engine](https://github.com/anchore/anchore-engine).  
Сканер уязвимостей для образов контейнеров и файловых систем, не сканирует git-репозиторий.  
Возможности:
- Сканирование содержимого образа контейнера или файловой системы для поиска известных уязвимостей.
- Обнаруживает уязвимости для распространенных систем
- Обнаруживает уязвимости в пакетах языков программирования
- Поддерживает форматы образов [Docker](https://www.docker.com/), [OCI](https://opencontainers.org/) и [SingularityCE](https://github.com/sylabs/singularity)

Из коробки работает с другим инструментом от Anchore - [syft](https://github.com/anchore/syft) (генератор SBOM), но так же воспринимает форматы [SPDX](https://spdx.dev/) [CycloneDX](https://cyclonedx.org/)

Так же можно обогатить базу уязвимостей связкой [grype](https://github.com/anchore/grype) + [grype-db](https://github.com/anchore/grype-db) + [vunnel](https://github.com/anchore/vunnel).  
Раньше была [интеграция с Gitlab](https://docs.gitlab.com/ee/user/application_security/container_scanning/), сейчас не поддерживается.

#### dockle
[dockle](https://github.com/goodwithtech/dockle) - линтер \ сканер.  
Ищет CVE в ПО образа, проверяет корректность и безопасность конкретного образа, анализируя его слои и конфигурацию.  
Используется CIS Benchmark.  
Работает как отдельно, так и интегрируется с GitLab CI, Jenkins.

#### docker-bench-security
[docker-bench-security](https://github.com/docker/docker-bench-security) - набор bash скриптов для проверки безопасности конфигурации образов и контейнеров. docker-bench-security больше рассчитан на интеграцию, standalone сканирование в целом есть, но не без страданий в работе.

Могут быть проблемы с интерпретацией вывода, так как используется CIS Benchmark.  
Давно не обновлялся, последняя версия от 20 декабря 2023 г.

#### Clair
[Clair](https://github.com/quay/clair) - проект для статического анализа уязвимостей в контейнерах приложений (в настоящее время включает [OCI](https://opencontainers.org/) и [Docker](https://www.docker.com/)).  
Клиенты используют API Clair для индексации своих образов контейнеров и затем могут сопоставить их с известными уязвимостями.

Проблемно разворачивается, не очень удобен в использовании.

#### Harbor
[Harbor](https://github.com/goharbor/harbor) - open source Docker Registry с безопасностью «из коробки». Под капотом уже имеет trivy, всё круто-здорово, но это Registry, так что не подходит.  

#### Dagda
[Dagda](https://github.com/eliasgranderubio/dagda) - инструмент для статического анализа известных уязвимостей,  вредоносного ПО и других  угроз в  образах/контейнерах, а также для мониторинга демона docker и запущенных docker контейнеров для выявления аномальных действий.
Давно не обновлялся, последний релиз Jul 27, 2021

#### Trivy
[Trivy](https://github.com/aquasecurity/trivy) - довольно известный сканер от Aqua Security. Находит уязвимые версии софта в зависимостях, секреты, мисконфиги.  
Из удобств: можно использовать как standalone сканирование, например проверить репозиторий целиком, так и интегрировать в CI/CD (GitLab CI, GitHub Actions). Есть плагин для VSCode и плагин для интеграции с Vulners.

#### Итоговый выбор
В итоге выбор пал на Trivy. Функционал trivy позволяет запускать его автоматизированно, сканировать как контейнеры, так и git-репозитории.  
Данные будут передаваться в одно место для хранения\агрегации\обработки, поэтому не требуется использовать сложный и комплексный инструмент, вроде Dagda или Clair.

Плагин [trivy-plugin-vulners-db](https://github.com/vulnersCom/trivy-plugin-vulners-db) позволяет обогащать данными сам отчёт (EPSS, CVSS, наличие эксплоитов, экслуатация "in-the-wild", работает так себе, но есть).
Плагин требует API-ключ для получения обновления баз, однако он запрашивается только при загрузке, а не каждом сканировании.

## Превозмогая трудности
### Общие проблемы
Но то, что неплохо работает в небольшом скоупе, может очень плохо масштабироваться. Либо делать это с компромиссами.
Сравнительное удобство Trivy имеет ряд своих компромиссов, одним из которых является [boltDB](https://github.com/boltdb/bolt). Сам boltDB является хранилищем на основе "ключ-значение", и из-за его ограничений параллельная работа Trivy по сути не работает.

Цитата [из документации](https://aquasecurity.github.io/trivy/v0.56/docs/references/troubleshooting/#running-in-parallel-takes-same-time-as-series-run): 
> Bolt obtains a file lock on the data file so multiple processes cannot open the same database at the same time. Opening an already open Bolt database will cause it to hang until the other process closes it.

Из коробки Trivy нельзя скормить список образов, но в целом это тоже решается, пускай и циклом на Bash.
А так же в дальнейшем вскрылся неприятный нюанс: плагин trivy-plugin-vulners-db EPSS не отдаёт. Этот вопрос решаем, но нюанс неприятный.

### Прикладное велосипедостроение
#### Получаем образы и тэги 
Объемы большие, поэтому просто сканировать по очереди будет долго. Образы будем пуллить. Для начала нам нужно получить список образов с последним тэгом:
```bash
REGISTRY_HOST="registry.exampledomain.ru"
NEXUS_HOST="nexus.exampledomain.ru"
IMAGES_LIST="images.list"
IMAGES_LIST_TAGGED="images_tagged.list"
TAGGED_COUNTER=0
DATE=$(date +'%Y-%m-%d')

curl -s -X GET -u "$REGISTRY_USER:$REGISTRY_PASSWORD" https://$REGISTRY_HOST/v2/_catalog | jq -r '.[].[]' > $IMAGES_LIST
IMAGE_QUANTITY=$(cat $IMAGES_LIST | wc -l)

get_tags(){
  curl -s -X GET -u "$REGISTRY_USER:$REGISTRY_PASSWORD" \
    "https://$NEXUS_HOST/service/rest/v1/search?repository=reponame&name=$1" \
    -H 'accept: application/json' | \
    jq -r '.items[] | .version as $version | .assets[] | {version: $version, lastModified: .lastModified}' | \
    jq -r -s 'sort_by(.lastModified) | last | .version'
}


for var in $(cat $IMAGES_LIST); do
    TAGGED_COUNTER=$((TAGGED_COUNTER + 1)) 
  status_code=0
  while [[ "$status_code" == "200" ]]; do
    status_code=$(curl -I -k -s "REGISTRY_HOST" | head -n 1 | cut -d ' ' -f 2)
    if [[ "$status_code" == "200" ]]; then
          echo "$(date): Tagged $TAGGED_COUNTER / $IMAGE_QUANTITY"
        echo "$(date): Tag $REGISTRY_HOST/$var"
        echo $var:$(get_tags $var) >> $IMAGES_LIST_TAGGED
    else
      echo "$(date): Repo status: $(status_code). Wait 30m."
      sleep 1800   
    fi
  done
done
```

После по получившемуся списку мы проходим `docker pull`. Не забываем, что для `docker login` у в `$HOME/.docker/config.json` нас должен лежать токен.
```bash
REGISTRY_HOST="registry.exampledomain.ru"
IMAGES_LIST_TAGGED="images_tagged.list"
DATE=$(date +'%Y-%m-%d')

docker login $REGISTRY_HOST

if [[ $? -ne 0 ]]; then
  echo "Docker login не удался."
  exit 1
fi

# Check registry host availability
check_registry() {
  local status_code
  status_code=$(curl -I -k -s "https://$REGISTRY_HOST" | head -n 1 | cut -d ' ' -f 2)
  if [[ "$status_code" == "400" ]]; then
    return 0
  else
    echo "$(date): Repo status: $status_code. Wait 30m."
    sleep 1800
    return 1
  fi
}

# Проверка наличия файла $IMAGES_LIST_TAGGED
if [[ ! -f $IMAGES_LIST_TAGGED ]]; then
  echo "Файл $IMAGE_LIST_TAGGED не найден."
  exit 1
fi

# Параллельная загрузка образов
pull_image() {
  local image="$1"

  # Ожидание доступности registry
  until check_registry; do
    echo "$(date): registry unavailable. Re-check  in 30 min."
  done

  echo "$(date): Pull: $REGISTRY_HOST/$image"
  docker pull "$REGISTRY_HOST/$image" >> pull.log 2>> err.log
}

export -f pull_image
export REGISTRY_HOST

cat "$IMAGES_LIST_TAGGED" | parallel -j 4 --bar pull_image {}
```

#### Сканируем
Список тэгированных образов получен. Осталось их скормить  
```bash
IMAGES_LOCAL_LIST="images_local.lst"
SCANNED_COUNTER=0
DATE=$(date +'%Y-%m-%d')

# Temp folder
mkdir -p tmp/
tmp_dir="tmp"

trivy_log_name=trivy.json

trivy vulners-db --api-key $VULNERS_API_KEY

docker images -a -q > $IMAGES_LOCAL_LIST
IMAGE_QUANTITY=$(cat $IMAGES_LOCAL_LIST| wc -l)

for var in $(cat $IMAGES_LOCAL_LIST); do
    SCANNED_COUNTER=$((SCANNED_COUNTER + 1))
    echo "$(date): Vulnerabilities search in image: $var"
    echo "$(date): Scan $SCANNED_COUNTER / $IMAGE_QUANTITY"
    reponame="${var//\//_}"
    /usr/bin/trivy -q image $var --ignore-policy policy.rego --scanners vuln  --format json -o $tmp_dir/$reponame.json
done
```

##### Ignore Policy
>[!info]
>Тут стоит сделать небольшую сноску. Я использую опицию `--ignore-policy`, так как я дальше обрабатываю данные от плагина vulners. В не-CVE уязвимостях может быть описание в markdown, что вызывает проблемы. 
>Если вы в дальнейшем не планируете обрабатывать его, то можно ignore policy не использовать.

Пример Ignore Policy:
```go
package trivy

# По умолчанию все уязвимости разрешены
default ignore = false

# Игнорируем уязвимости с идентификаторами, начинающимися на "GHSA", "DLA" или "TEMP"
ignore {
    input.VulnerabilityID != null
    startswith(input.VulnerabilityID, "GHSA")
}

ignore {
    input.VulnerabilityID != null
    startswith(input.VulnerabilityID, "DLA")
}

ignore {
    input.VulnerabilityID != null
    startswith(input.VulnerabilityID, "TEMP")
}

ignore {
    input.VulnerabilityID != null
    startswith(input.VulnerabilityID, "NSWG")
}

ignore {
    input.VulnerabilityID != null
    startswith(input.VulnerabilityID, "DSA")
}


```

#### Чистим и обогащаем
На выходе сканирования мы получаем довольно массивные JSON-файлы с множеством полезной и не очень информации. Референсные ссылки,  информацию о слоях и прочую крайне полезную информацию отправлять в какой-то коллектор смысла нет. А так же вспоминаем тот факт, что EPSS нам нужно добавить самостоятельно.

Так что чистим и обогащаем наши репорты. Для начала нужно отформатировать Description, которое получаем от плагина vulners:
```bash
tmp_dir="./tmp"
json_dir="json"
jsonl_dir="jsonl"
cve_cache_file="./cve_cache.txt"
DATE=$(date +'%Y-%m-%d')

mkdir -p $json_dir
mkdir -p $jsonl_dir

# Удаление репортов без Vulnerabilities
echo "$(date): Clean Non-Description: Start"
find ./tmp -name "*.json" | parallel -j 4 '
  if ! jq -e ".Results[].Vulnerabilities? | select(. != null)" {} > /dev/null 2>&1; then
    rm {}
  fi
'
echo "$(date): Clean Non-Description: Done"

# Форматирование Description
echo "$(date): Format Description: Start"
for filename in $(ls $tmp_dir); do
	jq '.Results[].Vulnerabilities[]?.Description |= if type == "string" and startswith("{") then (try fromjson catch .) else . end' "$tmp_dir"/"$filename"  > "$tmp_dir"/"$filename".tmp && mv "$tmp_dir"/"$filename".tmp "$tmp_dir"/"$filename"
done

# Костыль, репорты без Description не обрабатываются, но создаётся временный файл
rm tmp/*.json.tmp 2>/dev/null
echo "$(date): Format Description: Done" 
```
Description исправлен, теперь нужно убрать лишние поля, добавить нужное (можно добавить свои поля). Так же добавляем и EPSS из API FIRST:
```bash
# Форматирование тела JSON
echo "$(date): Format JSON Body: Start" 

# Форматирование и обрезка лишнего
for filename in $(ls $tmp_dir); do
  jq '{
    SchemaVersion,
    RepoTags: .Metadata.RepoTags[0],
    RepoDigests: .Metadata.RepoDigests[],
    Vulnerabilities: [
      .Results[].Vulnerabilities[]? | {
      Title,
      VulnerabilityID,
      Severity,
      PkgName,
      PkgPath,
      InstalledVersion,
      FixedVersion,
      Status,
      Cvss2_Score: (if (.Description | type) == "object" then .Description.Cvss2?.Score else null end),
      Cvss3_Score: (if (.Description | type) == "object" then .Description.Cvss3?.Score else null end),
      Epss,
      Epss_percent,
      VulnersScore: (if (.Description | type) == "object" then .Description.VulnersScore?.Value else null end),
      WildExploited: (if (.Description | type) == "object" then .Description.WildExploited else null end),
      ExploitsCount: (if (.Description | type) == "object" then .Description.ExploitsCount else null end),
      Vulners_Description: (if (.Description | type) == "object" then .Description.Description else null end),
      Href: (if (.Description | type) == "object" then .Description.Href else null end),
    }
  ] | unique
}' "$tmp_dir"/"$filename"  > "$tmp_dir"/"$filename".tmp && mv "$tmp_dir"/"$filename".tmp "$tmp_dir"/"$filename"
done

echo "$(date): Format JSON Body: Done" 

echo "$(date): Enrich process: Start"
# Создаем кэш-файл, если он не существует
touch "$cve_cache_file"

# Функция для получения EPSS с кэшированием
fetch_epss() {
  local cve_id=$1
  local cached=$(grep -E "^$cve_id\s" "$cve_cache_file" | awk '{print $2}')

  if [ -n "$cached" ]; then
    echo "$cached"
  else
    local api_response=$(curl -s "https://api.first.org/data/v1/epss?cve=$cve_id")
    local epss_value=$(echo "$api_response" | jq -r '.data[]?.epss // "null"')
    echo "$cve_id $epss_value" >> "$cve_cache_file"
    echo "$epss_value"
  fi
}

# Функция обработки одного файла
process_file() {
  local filename=$1
  local results=()

  # Проверяем, что файл существует
  if [ ! -f "$filename" ]; then
    echo "Ошибка: файл $filename не найден."
    return
  fi

  # Извлекаем данные одним вызовом jq
  metadata=$(jq -r '{
    cve_ids: [.Vulnerabilities[].VulnerabilityID // empty],
    repotags: (.RepoTags // empty),
    repodigests: (.RepoDigests // empty),
    namespace: (.RepoTags | if . then split("/")[1] else null end)
  }' "$filename")

  if [ $? -ne 0 ]; then
    echo "Ошибка обработки JSON файла $filename с помощью jq."
    return
  fi

  cve_ids=$(echo "$metadata" | jq -r '.cve_ids[]')
  repotags=$(echo "$metadata" | jq -r '.repotags')
  repodigests=$(echo "$metadata" | jq -r '.repodigests')
  namespace=$(echo "$metadata" | jq -r '.namespace')


  # Обработка каждого CVE ID
  for cve_id in $cve_ids; do
    # Получаем значение EPSS
    epss_value=$(fetch_epss "$cve_id")

    # Устанавливаем значение по умолчанию, если epss_value пустое или "null"
    if [[ -z "$epss_value" || "$epss_value" == "null" ]]; then
      epss_value=0
    fi

    # Рассчитываем процентное значение
    epss_percent=$(awk "BEGIN {printf \"%.2f\", $epss_value * 100}")

    # Генерация JSON 
    processed_json=$(jq --arg cve "$cve_id" \
      --arg Source "nexus" \
      --arg epss "$epss_value" \
      --arg Epss_percent "$epss_percent" \
      --arg repotags "$repotags" \
      --arg repodigests "$repodigests" '
        .Vulnerabilities[] | select(.VulnerabilityID == $cve) | 
        {"reposcanner": {
          Source: $Source,
          RepoTags: $repotags,
          RepoDigests: $repodigests,
          TeamId: $teamId,
          Title,
          VulnerabilityID: $cve,
          Severity: .Severity,
          PkgName: .PkgName,
          PkgPath: .PkgPath,
          InstalledVersion: .InstalledVersion,
          FixedVersion: .FixedVersion,
          Status: .Status,
          Cvss2_Score: .Cvss2_Score,
          Cvss3_Score: .Cvss3_Score,
          Epss: ($epss // "custom_null"),
          Epss_percent: $Epss_percent,
          VulnersScore: .VulnersScore,
          WildExploited: .WildExploited,
          ExploitsCount: .ExploitsCount,
          Vulners_Description: .Vulners_Description,
          Href: .Href      
        }}
      ' "$filename")

    if [ -n "$processed_json" ]; then
      results+=("$processed_json")
    else
      echo "Не удалось обработать данные для $cve_id."
    fi
  done


  # Объединяем результаты в один JSON
  if [ ${#results[@]} -eq 0 ]; then
    echo "Не найдено результатов для обработки файла $filename."
  else
    final_output=$(printf '%s\n' "${results[@]}" | jq -s '.')
    output_file="$json_dir/$(basename "$filename" .json).json"
    echo "$final_output" > "$output_file"
  fi
}

# Параллельная обработка всех файлов
export -f fetch_epss process_file
export json_dir cve_cache_file
find "$tmp_dir" -type f | parallel -j "$(nproc)" process_file {}
echo "$(date): Enrich process: Done"
```

Дальнейшие действия уже зависят от того, куда мы будем полученные репорты отправлять. Например, если нам нужен JSONL, мы можем с помощью jq репорты без проблем преобразовать:
```bash
# JSON to JSONL
echo "$(date): Convert JSON to JSONL: Start" 
# Читаем исходный JSON из файла
for filename in "$json_dir"/*; do

    # преобразовать JSON в JSONL 
    jq -c '.[]' $filename > $jsonl_dir/$(basename "$filename" .json).jsonl
done
echo "$(date): Convert JSON to JSONL: Done" 
```

## Итоги
Образы спуллены и отсканированы, репорты отформатированы и обмазаны EPSS. Дальше можно отправлять всё в коллектор, будь то Elastic, Graylog, DataDog, в какое-то кастомное решение или просто ~~читать перед сном~~ визуализировать в чём-то.

Хорошее ли это решение? Скорей нет. Это громадный и медленный костыль с кучей точек отказа.  
Жизнеспособное ли это решение? В целом, да. Если собирать и обрабатывать репорты где-то, с этим можно работать. Если денег нет, а скоуп небольшой, особых проблем быть не должно.

И самое главное: если нет процесса patch management, то никакие сканеры ситуацию не исправят. 

---
[Habr: Trivy: вредные советы по скрытию уязвимостей](https://habr.com/ru/companies/swordfish_security/articles/822705/)  
[Trivy Documentation](https://aquasecurity.github.io/trivy/v0.56/)  
[Github.com: Trivy](https://github.com/aquasecurity/trivy)  
[Github: trivy-plugin-vulners-db](https://github.com/vulnersCom/trivy-plugin-vulners-db)   
[FIRST API v1](https://api.first.org/)   

