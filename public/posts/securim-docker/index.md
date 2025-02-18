# Secure Docker


Когда речь заходит про безопасность docker, то обычно сразу всплывает всякий сбор логов, мониторинг и ~~&#34;надо было ставить линукс&#34;~~ &#34;давайте запихаем агент Wazuh&#34;. 
Однако, как с вешалки начинается театр, так жизненный путь контейнера начинается с dockerfile. C них и начнем. 
&lt;!--more--&gt;
Из простого: какой-нибудь линтер:
Для ленивых -[fromlatest.io](https://www.fromlatest.io)
Поправить синтаксис, теги, убрать всякий deprecated и вообще держать всё в удобочитаемом виде дело важное.

## [hadolint](https://github.com/hadolint/hadolint)
Локальный линтер, который можно запускать как в установленном виде, так и из докера. Может проверять как локальные докерфайлы, так и из registry.  Проверки репозитория нет, так что при необходимости придётся решать это костылями.

[Репозиторий](https://github.com/hadolint/hadolint)  
[Web-версия](https://hadolint.github.io/hadolint/)  

---

Сканеры уязвимости:
## [trivy](https://github.com/aquasecurity/trivy)
trivy - простой, но хороший сканер уязвимостей. Находит уязвимые версии софта в зависимостях, секреты, мисконфиги.
Из удобств: можно использовать как standalone сканирование, например проверить репозиторий целиком, так и интегрировать в CI/CD (Jenkins, GitLab CI, GitHub Actions). Есть плагин для VSCode.

[Репозиторий](https://github.com/aquasecurity/trivy)  
[Плагин для VSCode](https://github.com/aquasecurity/trivy-vscode-extension)  

## [docker-bench-security](https://github.com/docker/docker-bench-security)
Набор bash скриптов для проверки безопасности конфигурации образов и  контейнеров. docker-bench-security больше рассчитан на интеграцию, standalone сканирование в целом есть, но не без страданий в работе.

Могут быть проблемы с интерпритацией вывода, так как используется CIS Benchmark.

[Репозиторий](https://github.com/docker/docker-bench-security)

## [dockle](https://github.com/goodwithtech/dockle)
И линтер, и сканер.
Ищет CVE в ПО образа, проверяет корректность и безопасность конкретного образа, анализируя его слои и конфигурацию.
И вообще это лучший инструмент (по словам автора).

Работает как отдельно, так и интегрируется с GitLab CI, Jenkins.

[Репозиторий](https://github.com/goodwithtech/dockle)


И немного дополню это всё, так как только инструментами всё не решается.

[CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/),  
[Best Practice](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)  
[Top 20 Docker Security Best Practices](https://blog.aquasec.com/docker-security-best-practices)  

Да, кроме этого есть ещё и [clair](https://github.com/quay/clair), [Lynis](https://cisofy.com/lynis/), но первый довольно масштабный, и нужен не всем, а Lynis довольно своеобразен. 

Если вы не планируете интеграцию в CI/CD, то trivy отличный вариант. 
А в CI/CD можно и dockle или clair засунуть интегрировать.


---

> Author:   
> URL: http://localhost:1313/posts/securim-docker/  

