# Not So (Trivy)al. Как Использовать Trivy На Большом Скоупе И Не Поехать Кукухой (Нет)


Стояла задача простая и ясная как летний день - сканировать Docker-образы в своём registry. 
Но по мере работы стали проявляться нюансы, которые как Газманов забирали этот ясный день:
- Тормозной и неудобный API Nexus (частная проблема, решается использованием Registry API);
- Большое количество и объем образов (3к&#43; образов и более 1.5Tb объема);
- Не все используют тэг latest;
- В роли сборщика репортов не подходил DefectDojo.

&lt;!--more--&gt;

## Intro
&gt; [!info]
&gt; Тут не будет готового решения, скорей концептуальное описание пройденного пути.
&gt; Так же примеры кода будут в виде shell-скриптов, так как с языками программирования у меня не всё просто, а так же многое завязано за jq.
&gt; В статье будет про Nexus в роли репозитория, но основные моменты могут быть применимы и к другим решениям.



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

Так же можно обогатить базу уязвимостей связкой [grype](https://github.com/anchore/grype) &#43; [grype-db](https://github.com/anchore/grype-db) &#43; [vunnel](https://github.com/anchore/vunnel).  
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

Плагин [trivy-plugin-vulners-db](https://github.com/vulnersCom/trivy-plugin-vulners-db) позволяет обогащать данными сам отчёт (EPSS, CVSS, наличие эксплоитов, экслуатация &#34;in-the-wild&#34;, работает так себе, но есть).
Плагин требует API-ключ для получения обновления баз, однако он запрашивается только при загрузке, а не каждом сканировании.

## Превозмогая трудности
### Общие проблемы
Но то, что неплохо работает в небольшом скоупе, может очень плохо масштабироваться. Либо делать это с компромиссами.
Сравнительное удобство Trivy имеет ряд своих компромиссов, одним из которых является [boltDB](https://github.com/boltdb/bolt). Сам boltDB является хранилищем на основе &#34;ключ-значение&#34;, и из-за его ограничений параллельная работа Trivy по сути не работает.

Цитата [из документации](https://aquasecurity.github.io/trivy/v0.56/docs/references/troubleshooting/#running-in-parallel-takes-same-time-as-series-run): 
&gt; Bolt obtains a file lock on the data file so multiple processes cannot open the same database at the same time. Opening an already open Bolt database will cause it to hang until the other process closes it.

Из коробки Trivy нельзя скормить список образов, но в целом это тоже решается, пускай и циклом на Bash.
А так же в дальнейшем вскрылся неприятный нюанс: плагин trivy-plugin-vulners-db EPSS не отдаёт. Этот вопрос решаем, но нюанс неприятный.

### Прикладное велосипедостроение
#### Получаем образы и тэги 
Объемы большие, поэтому просто сканировать по очереди будет долго. Образы будем пуллить. Для начала нам нужно получить список образов с последним тэгом:
```bash
REGISTRY_HOST=&#34;registry.exampledomain.ru&#34;
NEXUS_HOST=&#34;nexus.exampledomain.ru&#34;
IMAGES_LIST=&#34;images.list&#34;
IMAGES_LIST_TAGGED=&#34;images_tagged.list&#34;
TAGGED_COUNTER=0
DATE=$(date &#43;&#39;%Y-%m-%d&#39;)

curl -s -X GET -u &#34;$REGISTRY_USER:$REGISTRY_PASSWORD&#34; https://$REGISTRY_HOST/v2/_catalog | jq -r &#39;.[].[]&#39; &gt; $IMAGES_LIST
IMAGE_QUANTITY=$(cat $IMAGES_LIST | wc -l)

get_tags(){
  curl -s -X GET -u &#34;$REGISTRY_USER:$REGISTRY_PASSWORD&#34; \
    &#34;https://$NEXUS_HOST/service/rest/v1/search?repository=reponame&amp;name=$1&#34; \
    -H &#39;accept: application/json&#39; | \
    jq -r &#39;.items[] | .version as $version | .assets[] | {version: $version, lastModified: .lastModified}&#39; | \
    jq -r -s &#39;sort_by(.lastModified) | last | .version&#39;
}


for var in $(cat $IMAGES_LIST); do
    TAGGED_COUNTER=$((TAGGED_COUNTER &#43; 1)) 
  status_code=0
  while [[ &#34;$status_code&#34; == &#34;200&#34; ]]; do
    status_code=$(curl -I -k -s &#34;REGISTRY_HOST&#34; | head -n 1 | cut -d &#39; &#39; -f 2)
    if [[ &#34;$status_code&#34; == &#34;200&#34; ]]; then
          echo &#34;$(date): Tagged $TAGGED_COUNTER / $IMAGE_QUANTITY&#34;
        echo &#34;$(date): Tag $REGISTRY_HOST/$var&#34;
        echo $var:$(get_tags $var) &gt;&gt; $IMAGES_LIST_TAGGED
    else
      echo &#34;$(date): Repo status: $(status_code). Wait 30m.&#34;
      sleep 1800   
    fi
  done
done
```

После по получившемуся списку мы проходим `docker pull`. Не забываем, что для `docker login` у в `$HOME/.docker/config.json` нас должен лежать токен.
```bash
REGISTRY_HOST=&#34;registry.exampledomain.ru&#34;
IMAGES_LIST_TAGGED=&#34;images_tagged.list&#34;
DATE=$(date &#43;&#39;%Y-%m-%d&#39;)

docker login $REGISTRY_HOST

if [[ $? -ne 0 ]]; then
  echo &#34;Docker login не удался.&#34;
  exit 1
fi

# Check registry host availability
check_registry() {
  local status_code
  status_code=$(curl -I -k -s &#34;https://$REGISTRY_HOST&#34; | head -n 1 | cut -d &#39; &#39; -f 2)
  if [[ &#34;$status_code&#34; == &#34;400&#34; ]]; then
    return 0
  else
    echo &#34;$(date): Repo status: $status_code. Wait 30m.&#34;
    sleep 1800
    return 1
  fi
}

# Проверка наличия файла $IMAGES_LIST_TAGGED
if [[ ! -f $IMAGES_LIST_TAGGED ]]; then
  echo &#34;Файл $IMAGE_LIST_TAGGED не найден.&#34;
  exit 1
fi

# Параллельная загрузка образов
pull_image() {
  local image=&#34;$1&#34;

  # Ожидание доступности registry
  until check_registry; do
    echo &#34;$(date): registry unavailable. Re-check  in 30 min.&#34;
  done

  echo &#34;$(date): Pull: $REGISTRY_HOST/$image&#34;
  docker pull &#34;$REGISTRY_HOST/$image&#34; &gt;&gt; pull.log 2&gt;&gt; err.log
}

export -f pull_image
export REGISTRY_HOST

cat &#34;$IMAGES_LIST_TAGGED&#34; | parallel -j 4 --bar pull_image {}
```

#### Сканируем
Список тэгированных образов получен. Осталось их скормить  
```bash
IMAGES_LOCAL_LIST=&#34;images_local.lst&#34;
SCANNED_COUNTER=0
DATE=$(date &#43;&#39;%Y-%m-%d&#39;)

# Temp folder
mkdir -p tmp/
tmp_dir=&#34;tmp&#34;

trivy_log_name=trivy.json

trivy vulners-db --api-key $VULNERS_API_KEY

docker images -a -q &gt; $IMAGES_LOCAL_LIST
IMAGE_QUANTITY=$(cat $IMAGES_LOCAL_LIST| wc -l)

for var in $(cat $IMAGES_LOCAL_LIST); do
    SCANNED_COUNTER=$((SCANNED_COUNTER &#43; 1))
    echo &#34;$(date): Vulnerabilities search in image: $var&#34;
    echo &#34;$(date): Scan $SCANNED_COUNTER / $IMAGE_QUANTITY&#34;
    reponame=&#34;${var//\//_}&#34;
    /usr/bin/trivy -q image $var --ignore-policy policy.rego --scanners vuln  --format json -o $tmp_dir/$reponame.json
done
```

##### Ignore Policy
&gt;[!info]
&gt;Тут стоит сделать небольшую сноску. Я использую опицию `--ignore-policy`, так как я дальше обрабатываю данные от плагина vulners. В не-CVE уязвимостях может быть описание в markdown, что вызывает проблемы. 
&gt;Если вы в дальнейшем не планируете обрабатывать его, то можно ignore policy не использовать.

Пример Ignore Policy:
```go
package trivy

# По умолчанию все уязвимости разрешены
default ignore = false

# Игнорируем уязвимости с идентификаторами, начинающимися на &#34;GHSA&#34;, &#34;DLA&#34; или &#34;TEMP&#34;
ignore {
    input.VulnerabilityID != null
    startswith(input.VulnerabilityID, &#34;GHSA&#34;)
}

ignore {
    input.VulnerabilityID != null
    startswith(input.VulnerabilityID, &#34;DLA&#34;)
}

ignore {
    input.VulnerabilityID != null
    startswith(input.VulnerabilityID, &#34;TEMP&#34;)
}

ignore {
    input.VulnerabilityID != null
    startswith(input.VulnerabilityID, &#34;NSWG&#34;)
}

ignore {
    input.VulnerabilityID != null
    startswith(input.VulnerabilityID, &#34;DSA&#34;)
}


```

#### Чистим и обогащаем
На выходе сканирования мы получаем довольно массивные JSON-файлы с множеством полезной и не очень информации. Референсные ссылки,  информацию о слоях и прочую крайне полезную информацию отправлять в какой-то коллектор смысла нет. А так же вспоминаем тот факт, что EPSS нам нужно добавить самостоятельно.

Так что чистим и обогащаем наши репорты. Для начала нужно отформатировать Description, которое получаем от плагина vulners:
```bash
tmp_dir=&#34;./tmp&#34;
json_dir=&#34;json&#34;
jsonl_dir=&#34;jsonl&#34;
cve_cache_file=&#34;./cve_cache.txt&#34;
DATE=$(date &#43;&#39;%Y-%m-%d&#39;)

mkdir -p $json_dir
mkdir -p $jsonl_dir

# Удаление репортов без Vulnerabilities
echo &#34;$(date): Clean Non-Description: Start&#34;
find ./tmp -name &#34;*.json&#34; | parallel -j 4 &#39;
  if ! jq -e &#34;.Results[].Vulnerabilities? | select(. != null)&#34; {} &gt; /dev/null 2&gt;&amp;1; then
    rm {}
  fi
&#39;
echo &#34;$(date): Clean Non-Description: Done&#34;

# Форматирование Description
echo &#34;$(date): Format Description: Start&#34;
for filename in $(ls $tmp_dir); do
	jq &#39;.Results[].Vulnerabilities[]?.Description |= if type == &#34;string&#34; and startswith(&#34;{&#34;) then (try fromjson catch .) else . end&#39; &#34;$tmp_dir&#34;/&#34;$filename&#34;  &gt; &#34;$tmp_dir&#34;/&#34;$filename&#34;.tmp &amp;&amp; mv &#34;$tmp_dir&#34;/&#34;$filename&#34;.tmp &#34;$tmp_dir&#34;/&#34;$filename&#34;
done

# Костыль, репорты без Description не обрабатываются, но создаётся временный файл
rm tmp/*.json.tmp 2&gt;/dev/null
echo &#34;$(date): Format Description: Done&#34; 
```
Description исправлен, теперь нужно убрать лишние поля, добавить нужное (можно добавить свои поля). Так же добавляем и EPSS из API FIRST:
```bash
# Форматирование тела JSON
echo &#34;$(date): Format JSON Body: Start&#34; 

# Форматирование и обрезка лишнего
for filename in $(ls $tmp_dir); do
  jq &#39;{
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
      Cvss2_Score: (if (.Description | type) == &#34;object&#34; then .Description.Cvss2?.Score else null end),
      Cvss3_Score: (if (.Description | type) == &#34;object&#34; then .Description.Cvss3?.Score else null end),
      Epss,
      Epss_percent,
      VulnersScore: (if (.Description | type) == &#34;object&#34; then .Description.VulnersScore?.Value else null end),
      WildExploited: (if (.Description | type) == &#34;object&#34; then .Description.WildExploited else null end),
      ExploitsCount: (if (.Description | type) == &#34;object&#34; then .Description.ExploitsCount else null end),
      Vulners_Description: (if (.Description | type) == &#34;object&#34; then .Description.Description else null end),
      Href: (if (.Description | type) == &#34;object&#34; then .Description.Href else null end),
    }
  ] | unique
}&#39; &#34;$tmp_dir&#34;/&#34;$filename&#34;  &gt; &#34;$tmp_dir&#34;/&#34;$filename&#34;.tmp &amp;&amp; mv &#34;$tmp_dir&#34;/&#34;$filename&#34;.tmp &#34;$tmp_dir&#34;/&#34;$filename&#34;
done

echo &#34;$(date): Format JSON Body: Done&#34; 

echo &#34;$(date): Enrich process: Start&#34;
# Создаем кэш-файл, если он не существует
touch &#34;$cve_cache_file&#34;

# Функция для получения EPSS с кэшированием
fetch_epss() {
  local cve_id=$1
  local cached=$(grep -E &#34;^$cve_id\s&#34; &#34;$cve_cache_file&#34; | awk &#39;{print $2}&#39;)

  if [ -n &#34;$cached&#34; ]; then
    echo &#34;$cached&#34;
  else
    local api_response=$(curl -s &#34;https://api.first.org/data/v1/epss?cve=$cve_id&#34;)
    local epss_value=$(echo &#34;$api_response&#34; | jq -r &#39;.data[]?.epss // &#34;null&#34;&#39;)
    echo &#34;$cve_id $epss_value&#34; &gt;&gt; &#34;$cve_cache_file&#34;
    echo &#34;$epss_value&#34;
  fi
}

# Функция обработки одного файла
process_file() {
  local filename=$1
  local results=()

  # Проверяем, что файл существует
  if [ ! -f &#34;$filename&#34; ]; then
    echo &#34;Ошибка: файл $filename не найден.&#34;
    return
  fi

  # Извлекаем данные одним вызовом jq
  metadata=$(jq -r &#39;{
    cve_ids: [.Vulnerabilities[].VulnerabilityID // empty],
    repotags: (.RepoTags // empty),
    repodigests: (.RepoDigests // empty),
    namespace: (.RepoTags | if . then split(&#34;/&#34;)[1] else null end)
  }&#39; &#34;$filename&#34;)

  if [ $? -ne 0 ]; then
    echo &#34;Ошибка обработки JSON файла $filename с помощью jq.&#34;
    return
  fi

  cve_ids=$(echo &#34;$metadata&#34; | jq -r &#39;.cve_ids[]&#39;)
  repotags=$(echo &#34;$metadata&#34; | jq -r &#39;.repotags&#39;)
  repodigests=$(echo &#34;$metadata&#34; | jq -r &#39;.repodigests&#39;)
  namespace=$(echo &#34;$metadata&#34; | jq -r &#39;.namespace&#39;)


  # Обработка каждого CVE ID
  for cve_id in $cve_ids; do
    # Получаем значение EPSS
    epss_value=$(fetch_epss &#34;$cve_id&#34;)

    # Устанавливаем значение по умолчанию, если epss_value пустое или &#34;null&#34;
    if [[ -z &#34;$epss_value&#34; || &#34;$epss_value&#34; == &#34;null&#34; ]]; then
      epss_value=0
    fi

    # Рассчитываем процентное значение
    epss_percent=$(awk &#34;BEGIN {printf \&#34;%.2f\&#34;, $epss_value * 100}&#34;)

    # Генерация JSON 
    processed_json=$(jq --arg cve &#34;$cve_id&#34; \
      --arg Source &#34;nexus&#34; \
      --arg epss &#34;$epss_value&#34; \
      --arg Epss_percent &#34;$epss_percent&#34; \
      --arg repotags &#34;$repotags&#34; \
      --arg repodigests &#34;$repodigests&#34; &#39;
        .Vulnerabilities[] | select(.VulnerabilityID == $cve) | 
        {&#34;reposcanner&#34;: {
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
          Epss: ($epss // &#34;custom_null&#34;),
          Epss_percent: $Epss_percent,
          VulnersScore: .VulnersScore,
          WildExploited: .WildExploited,
          ExploitsCount: .ExploitsCount,
          Vulners_Description: .Vulners_Description,
          Href: .Href      
        }}
      &#39; &#34;$filename&#34;)

    if [ -n &#34;$processed_json&#34; ]; then
      results&#43;=(&#34;$processed_json&#34;)
    else
      echo &#34;Не удалось обработать данные для $cve_id.&#34;
    fi
  done


  # Объединяем результаты в один JSON
  if [ ${#results[@]} -eq 0 ]; then
    echo &#34;Не найдено результатов для обработки файла $filename.&#34;
  else
    final_output=$(printf &#39;%s\n&#39; &#34;${results[@]}&#34; | jq -s &#39;.&#39;)
    output_file=&#34;$json_dir/$(basename &#34;$filename&#34; .json).json&#34;
    echo &#34;$final_output&#34; &gt; &#34;$output_file&#34;
  fi
}

# Параллельная обработка всех файлов
export -f fetch_epss process_file
export json_dir cve_cache_file
find &#34;$tmp_dir&#34; -type f | parallel -j &#34;$(nproc)&#34; process_file {}
echo &#34;$(date): Enrich process: Done&#34;
```

Дальнейшие действия уже зависят от того, куда мы будем полученные репорты отправлять. Например, если нам нужен JSONL, мы можем с помощью jq репорты без проблем преобразовать:
```bash
# JSON to JSONL
echo &#34;$(date): Convert JSON to JSONL: Start&#34; 
# Читаем исходный JSON из файла
for filename in &#34;$json_dir&#34;/*; do

    # преобразовать JSON в JSONL 
    jq -c &#39;.[]&#39; $filename &gt; $jsonl_dir/$(basename &#34;$filename&#34; .json).jsonl
done
echo &#34;$(date): Convert JSON to JSONL: Done&#34; 
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



---

> Author:   
> URL: http://localhost:1313/posts/not_so_trivial/  

