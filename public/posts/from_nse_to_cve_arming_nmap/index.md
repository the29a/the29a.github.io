# От NSE До CVE. Вооружаем Nmap Для Поиска Уязвимостей


У nmap есть мощный инструмент, который незаслуженно обойден вниманием. И имя ему - NSE.  
Но сейчас мы не будем погружаться в глубины Lua, а рассмотрим те скрипты, которые нам уже доступны.

&lt;!--more--&gt;

## Немного о самих скриптах
На момент написания доступно 608 скриптов, но есть и скрипты написанные сообществом.  
Всего есть 14 категорий скриптов: `auth`, `broadcast`, `brute`, `default`, `discovery`, `dos`, `exploit`, `external`, `fuzzer`, `intrusive`, `malware`, `safe`, `version`, `vuln`.  
Но можно воспринимать их как своеобразные тэги, так как один и тот же скрипт может быть в нескольких категориях, например:
```lua
cat vulners.nse | grep categories 

categories = {&#34;vuln&#34;, &#34;safe&#34;, &#34;external&#34;, &#34;default&#34;}
```
Стандартная директория  размещения скриптов:  
`/usr/share/nmap/scripts/`

Обновление базы скриптов:
```bash
sudo nmap --script-updatedb 
```

## Примеры использования
При сканировании c опцией `-sV` (или её эквивалентом  `--script=default`) мы можем использовать скрипты из категории `default` :
```shell
nmap -sV -sC scanme.nmap.org
nmap -sV --script=default scanme.nmap.org
```

Так же можно использовать несколько скриптов совместно:
```shell
nmap --script nmap-vulners/,vulscan/ -sV scanme.nmap.org
```

## Сканирование с использованием скриптов
### vuln
Сканирование с использованием скриптов из из категории `vuln` :
```shell
nmap --script vuln -sV scamne.nmap.org 
```
Есть дополнительная опция `--script-args mincvss`, позволяющая указать минимальный уровень CVSS:
```
nmap --script vuln --script-args mincvss=4.9 -sV localhost  
```
&gt; Не совсем понятно, почему некоторые выделяют это как отдельный скрипт, хотя это именно использование категории скриптов.

### vulscan
[vulscan](https://github.com/scipag/vulscan) - скрипт представляющий больше исторический интерес, чем практический. Сам скрипт работоспособен, но фиды уязвимостей не обновлялись очень давно. Может быть актуально, если вы работаете с инфраструктурой пятнадцатилетней давности.   
Возможно, есть желающие разобраться с обновлением фидов, но насколько это будет целесообразно - вопрос открытый.  

Пример использования:
```shell
nmap -sV --script vulscan -sV scamne.nmap.org 
```
Из  особенностей отличался возможностью сканирования по определенной базе уявзимостей, например exploitdb:
```
nmap -sV --script vulscan --script-args vulscandb=exploitdb.csv scamne.nmap.org 
```

Судя по [сайту автора](https://www.computec.ch/), он немного выгорел, и оставил все наработки и ресурсы как исторический памятник.

### nmap-vulners
[nmap-vulners](https://github.com/vulnersCom/nmap-vulners) - скрипт от команды Vulners, использующий Vulners API,  и уже включен в стандартный набор скриптов nmap.  
У vulners не один скрипт, а целых три:  
- vulners.nse - основной скрипт, работающий на детекте версии.
- http-vulners-regex - скрипт сканирует HTTP-ответы и идентифицирует CPE для  программного обеспечения и может повысить эффективность основного скрипта.
- vulners_enterprise.nse - По сути, это vulners.nse с одним исключением: для его работы требуется API_KEY.

Пример использования:
```shell
nmap --script vulners -sV scanme.nmap.org
```
Есть дополнительная опция `--script-args mincvss`, позволяющая указать минимальный уровень CVSS:
```
nmap -sV --script vulners --script-args mincvss=5.0 scanme.nmap.org
```


### CVEScannerV2
[CVEScannerV2](https://github.com/scmanjarrez/CVEScannerV2) -  использует локальную базу о уязвимостях на основе данных [NVD](nvd.nist.gov/) и требует API-ключ, для первичной загрузки и обновления.  
Работает с детектом версий от nmap, и с детектом HTTP.

Пример использования:
```shell
nmap --script cvescannerv2 -sV scanme.nmap.org
```

## Расширяем возможности
При использовании скриптом для поиска уязвимостей мы можем обогатить сгенерированные отчёты о сканировании. Достаточно сохранять отчёт в `xml`, и с использованием [nmap-bootstrap-xsl](https://github.com/Haxxnet/nmap-bootstrap-xsl) или [nmap-formater](https://github.com/vdjagilev/nmap-formatter) конвертировать в удобный для чтения и хранения html-отчёт.  

Defect Dojo понимает [xml-формат отчётов](https://documentation.defectdojo.com/integrations/parsers/file/nmap/), так что мы можем использовать nmap (скрипт от Vulners в таком случае работает отлично) как полноценный сканер уязвимости с фронтендом в виде Defect Dojo. 

## Outro
С помощью nmap и NSE можно собрать небольшой сканер уязвимостей, останется выбрать нужные скрипты или вовсе ограничиться CVEScannerV2 и скриптами от vulners.  
Разумеется, это не покрывает все задачи, но при отсутствии ресурсов будет подспорьем в процессе работы.  

---
## Reference
### Nmap NSE Reference
[nmap.org: NSE](https://nmap.org/nsedoc/scripts/)  
[nmap.org: NSE Usage and Examples](https://nmap.org/book/nse-usage.html)  
[nmap.org: NSE Categories](https://nmap.org/nsedoc/categories/vuln.html)  

### Nmap NSE scripts
[Github.com NSE scripts from ivre](https://github.com/ivre/ivre/tree/master/patches/nmap/scripts)  
[Github.com: Nmap-NSE-scripts-collection](https://github.com/emadshanab/Nmap-NSE-scripts-collection)  
[Github.com: nmap-scripts](https://github.com/takeshixx/nmap-scripts)  


---

> Author: the29a  
> URL: http://localhost:1313/posts/from_nse_to_cve_arming_nmap/  

