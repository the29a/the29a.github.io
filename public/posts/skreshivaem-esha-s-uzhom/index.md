# Скрещиваем Ежа С Ужом. Заводим Репорты Nuclei В Defect Dojo


Сканерами уязвимостей пользуются многие, и в обороне и в атаке. Кто-то использует дорогой SaaS, кто-то использует opensource-решения вроде OpenVAS или даже SaaS на его основе.  
Казалось бы, можно использовать готовые решения, но мы пойдём другим путём и будем заводить репорты Nuclei в Defect Dojo.
Как раз, поддержка репортов есть, нужно только ей воспользоваться.
&lt;!--more--&gt;
Можно конечно использовать [Rengine](https://github.com/yogeshojha/rengine), и это бы решило часть проблем. Но задача стояла в сборе уязвимостей не только от nuclei, но и других инструментов, вроде [trivy](https://github.com/aquasecurity/trivy).

Так же не могу сказать, что я в восторге от Defect Dojo, но замены ему я не нашёл. По крайней мере из opensource.  

Отдельно отмечу, что это тестовый пример и на руководство на полноценное внедрение он не тянет.  
Рекомендую для тестов поднять заведомо уязвимый хост, чтобы удостоверится в работоспособности.  

## Используемые инструменты
Нам потребуются:  
docker-compose &gt;= 1.28
bash  
[Defect Dojo](https://github.com/DefectDojo/django-DefectDojo)  
[Nuclei](https://github.com/projectdiscovery/nuclei)  

### Defect Dojo
[Разворачиваем Defect Dojo](https://github.com/DefectDojo/django-DefectDojo#quick-start):
```
git clone https://github.com/DefectDojo/django-DefectDojo
cd django-DefectDojo
# Собираем
./dc-build.sh
# Запускаем (будем собирать так, но другие профили сборки можно посмотреть в документации https://github.com/DefectDojo/django-DefectDojo/blob/dev/readme-docs/DOCKER.md)
./dc-up.sh postgres-redis
# Или запускаем в фоне 
./dc-up-d.sh postgres-redis
# пароль пользователя admin будет в логах при запуске, так что будем грепать
docker-compose logs initializer | grep &#34;Admin password:&#34;
```

Если всё запустилось, то на [http://localhost:8080/](http://localhost:8080/) нас будет ждать Defect Dojo.  

Нам от него нужно:
1. Получить api-token пользователя
2. Создать product

#### 1. Получаем api-token
В правом верхнем углу нажимаем на &#34;человечка&#34;, нажимаем на API v2 Key. Готово. 

#### 2. Создаём product
В левой панели &#34;Products&#34; выбираем &#34;Add Product&#34;, где заполняем необходимые поля.
&gt;Важно, что в данном примере у нас будет имя `Nuclei Test Scan`, так как оно будет использоваться и далее.

В дальнейшем стоит создать отдельный Product Type и указывать его, но в данном случае будем использовать стандартный.  
Ещё нам потребуется product id, который мы можем увидеть в адресной строке. Он нам так же понадобится.  

### Nuclei
[Установить nuclei](https://github.com/projectdiscovery/nuclei#install-nuclei) можно [множеством способов](https://nuclei.projectdiscovery.io/nuclei/get-started/), но проще всего скачать архив с бинарником со [страницы релизов](https://github.com/projectdiscovery/nuclei/releases).

#### Подготовка
Перед началом сканирования стоит сделать отдельный конфиг, чтобы использовать только конкретные шаблоны для поиска CVE, уязвимостей и мисконфигов.
Так же стоит использовать дополнительные хидеры, чтобы используемый вами сканер можно было отделить от кучи других запросов.
И разумеется, выставляем допустимый `rate-limit`.

##### Пример конфига:
```
# Headers to include with all HTTP request
header:
  - &#39;X-sec-tools: secops/nuclei&#39;
  - &#39;User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) / nuclei&#39;

# Directory based template execution
templates:
  - cves/
  - vulnerabilities/
  - misconfiguration/

# Template Filters
tags: exposures,cve
severity: critical, high, medium

# Template Denylist
## Tag based exclusion
exclude-tags: info 

# Rate Limit configuration
rate-limit: 50
```
Сохраняем конфиг в файл `nuclei.conf` и оставляем его рядом с бинарником.  


#### Использование
В целом, у нас всё готово, но без скрипта запускать nuclei с множеством опций это сомнительное удовольствие.  
Так же, к этому моменту уже можно собрать список хостов для сканирования.  

Небольшие пояснения по скрипту:
- В секции `Defect Dojo` мы указываем хост, порт и токен. Так же идут эндпоинты api, их мы не меняем.
- Далее идёт время, обовление nuclei и его теплейтов (опционально, можно закомментить) и сам запуск nuclei с опциями.
- В секции `Create engagement` указываем необходимый product id (`product`), про который говорилось выше.
- И наконец секция Send result to defect dojo, где мы указываем `product_name`.

##### Пример скрипта:
```
#!/bin/bash

# Defect Dojo
defect_dojo_host=&#34;localhost&#34;
defect_dojo_port=&#34;8080&#34;
defect_dojo_auth_token=&#34;bb671699482fe665a021745e34671d2eb45eba5c&#34;
defect_dojo_api_import_scan=&#34;api/v2/import-scan&#34;
defect_dojo_api_engagement=&#34;api/v2/engagements&#34;

# Other VARs
date=$(date &#43;&#39;%Y-%m-%d&#39;)

# Check updates (may be useless in containers)
./nuclei -up
./nuclei -ut

# Run scan
./nuclei -l targets.txt -config ./nuclei.conf -j -o result.json

# Create engagement
curl -X POST &#34;http://$defect_dojo_host:$defect_dojo_port/$defect_dojo_api_engagement/&#34; \
-H &#34;Authorization: Token $defect_dojo_auth_token&#34; \
-F &#34;name=Nuclei $date&#34; \
-F &#34;product=2&#34; \
-F &#34;target_start=$date&#34; \
-F &#34;target_end=$date&#34;

# Send result to defect dojo
curl -X POST &#34;http://$defect_dojo_host:$defect_dojo_port/$defect_dojo_api_import_scan/&#34; \
  -H &#34;accept: application/json&#34; \
  -H &#34;Content-Type: multipart/form-data&#34; \
  -H &#34;Authorization: Token $defect_dojo_auth_token&#34; \
  -F &#34;minimum_severity=Info&#34; \
  -F &#34;active=true&#34; \
  -F &#34;verified=true&#34; \
  -F &#34;scan_type=Nuclei Scan&#34; \
  -F &#34;close_old_findings=false&#34; \
  -F &#34;push_to_jira=false&#34; \
  -F &#34;file=@./result.json&#34; \
  -F &#34;product_name=Nuclei Test Scan&#34; \
  -F &#34;scan_date=2023-05-02&#34; \
  -F &#34;engagement_name=Nuclei $date&#34;
```
Запускаем, смотрим Engagements в нашем продукте, и если всё отработало верно, то там нас будут ждать находки в виде CVE и\или уязвимостей.  

## Итог
Разумеется, есть куда стремится. Ручное указание product id это не удобно, решению не хватает гибкости.  
Но решение есть и оно работает, что уже хорошо.


---

> Author:   
> URL: http://localhost:1313/posts/skreshivaem-esha-s-uzhom/  

