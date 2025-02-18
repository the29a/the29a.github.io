# Пиджаки В Облаках. AWS Для ЦРУ.


AWS большой облачный провайдер, с множеством дата-центров, регионов, сервисов и множеством удобных вещей.  
Но очевидно, что AWS это не только public cloud и free tier для VPN, но и множество вещей, скрытых от обычного пользователя. 

&lt;!--more--&gt;

## Disclaimer
Данная статья не является &#34;срывом покровов&#34;, долгосрочным результатом работы кибер-разведки, или иной формой откровения. 
Это наглядный пример, что немалое количество информации можно получить просто почитав документацию и файлы конфигурации.   
Это применимо не только к основному продукту, но и к сторонним инструментам.  

Кроме того, это показывает что корпорации делают большие деньги не только на обычных пользователях, но и на больших государственных заказах. Так что немного посчитаем, сколько Amazon зарабатывает на них.

## Небольшой глосcарий
- AWS - Amazon Web Services
- CIA - ЦРУ, оно же Центральное Разведовательное Управление США
- IC - [Intelligence Community](https://en.wikipedia.org/wiki/United_States_Intelligence_Community), разведовательное сообщество, она же американская разведка 
- C2S - IC’s Commercial Cloud Services, коммерческие сервисы американской разведки 

## Intro
В AWS есть концепция под названием &#34;partition&#34; (раздел), которая объединяет несколько регионов. Классификация разделов описана в документации.

&gt; AWS группирует регионы в разделы. Каждый регион находится ровно в одном разделе, а каждый раздел имеет один или несколько регионов. Разделы имеют независимые экземпляры _AWS Identity and Access Management_ (IAM) и обеспечивают жесткую границу между регионами в разных разделах.
Коммерческие регионы AWS находятся в разделе aws, регионы в Китае - в разделе __aws-cn__, а регионы _AWS GovCloud_ - в разделе __aws-us-gov__.

Формат доменного имени эндпоинтов API различается в зависимости от раздела.

### Обзорная таблица
| AWS Partition code | AWS Partition Name        |
| ------------------ | ------------------------- |
| aws                | AWS Standart              |
| aws-ch             | AWS China                 |
| aws-us-gov         | AWS GovCloud (US)         |
| aws-iso            | AWS ISO (US) - Top Secret |
| aws-iso-b          | AWS ISOB (US) - Secret    |
| aws-iso-e          | AWS ISOE (Eu) - ?         |
| aws-iso-f          | AWS ISOF - ?              |

Как мы можем видеть, есть целых семь partition различной интересности.   
Давайте посмотрим, насколько глубока кроличья нора.  
![](../../static/20231016150826.png)

## Non Secret Partitions
### Стандартный partition AWS
| AWS Partition code | AWS Partition Name        |
| ------------------ | ------------------------- |
| aws                | AWS Standart              |

Это раздел, в котором находятся общедоступные регионы.   
Сюда включены все обычные коммерческие регионы, которые описаны в документации и доступны обычному пользователю.

#### Регионы
```go
const (
    AfSouth1RegionID     = &#34;af-south-1&#34;     // Africa (Cape Town).
    ApEast1RegionID      = &#34;ap-east-1&#34;      // Asia Pacific (Hong Kong).
    ApNortheast1RegionID = &#34;ap-northeast-1&#34; // Asia Pacific (Tokyo).
    ApNortheast2RegionID = &#34;ap-northeast-2&#34; // Asia Pacific (Seoul).
    ApNortheast3RegionID = &#34;ap-northeast-3&#34; // Asia Pacific (Osaka).
    ApSouth1RegionID     = &#34;ap-south-1&#34;     // Asia Pacific (Mumbai).
    ApSoutheast1RegionID = &#34;ap-southeast-1&#34; // Asia Pacific (Singapore).
    ApSoutheast2RegionID = &#34;ap-southeast-2&#34; // Asia Pacific (Sydney).
    ApSoutheast3RegionID = &#34;ap-southeast-3&#34; // Asia Pacific (Jakarta).
    CaCentral1RegionID   = &#34;ca-central-1&#34;   // Canada (Central).
    EuCentral1RegionID   = &#34;eu-central-1&#34;   // Europe (Frankfurt).
    EuNorth1RegionID     = &#34;eu-north-1&#34;     // Europe (Stockholm).
    EuSouth1RegionID     = &#34;eu-south-1&#34;     // Europe (Milan).
    EuWest1RegionID      = &#34;eu-west-1&#34;      // Europe (Ireland).
    EuWest2RegionID      = &#34;eu-west-2&#34;      // Europe (London).
    EuWest3RegionID      = &#34;eu-west-3&#34;      // Europe (Paris).
    MeSouth1RegionID     = &#34;me-south-1&#34;     // Middle East (Bahrain).
    SaEast1RegionID      = &#34;sa-east-1&#34;      // South America (Sao Paulo).
    UsEast1RegionID      = &#34;us-east-1&#34;      // US East (N. Virginia).
    UsEast2RegionID      = &#34;us-east-2&#34;      // US East (Ohio).
    UsWest1RegionID      = &#34;us-west-1&#34;      // US West (N. California).
    UsWest2RegionID      = &#34;us-west-2&#34;      // US West (Oregon).
)
```

### Partition AWS в Китае 
| AWS Partition code | AWS Partition Name        |
| ------------------ | ------------------------- |
| aws-cn             | AWS China                 |

Как следует из названия, это китайский регион AWS.  
__aws-cn__ имеет другой домен, www.amazonaws.cn, и _Amazon Resource Name_ (ARN) синтаксически влючая в arn имена cn: `aws–cn`  
Эти регионы не имеют прямой связи с AWS Global.  
Для открытия счета необходимо юридическое лицо, зарегистрированное в Китае.

#### Регионы
```go
const (
    CnNorth1RegionID     = &#34;cn-north-1&#34;     // China (Beijing).
    CnNorthwest1RegionID = &#34;cn-northwest-1&#34; // China (Ningxia).
)
```

Кстати, AWS проиграла в Китае судебный процесс по поводу товарного знака «AWS». Таким образом, «AWS», встречающееся в документе, не является товарным знаком, а просто аббревиатурой.  
[Amazon Web Services подает апелляцию после проигрыша в войне за товарные знаки AWS в Китае местному бизнесу Actionsoft](https://www.theregister.com/2021/01/05/aws_chinese_trademark/)  

### Partition AWS GovCloud (США)
| AWS Partition code | AWS Partition Name        |
| ------------------ | ------------------------- |
| aws-us-gov         | GovCloud (US)             |

__AWS GovCloud__ был создан для обслуживания потребностей федеральных, государственных и местных органов власти.  
Управляется гражданами США на территории США.  
Отсутствуют некоторые сервисы (вероятно, связанно с [ISO and CSA STAR Certified](https://aws.amazon.com/compliance/iso-certified/)).  
Использует конечные точки, специфичные для AWS GovCloud, которые находятся в открытом доступе в Интернете, но доступны только клиентам AWS GovCloud.  

#### Общее описание из endpoints
```json
{
    &#34;defaults&#34; : {
      &#34;hostname&#34; : &#34;{service}.{region}.{dnsSuffix}&#34;,
      &#34;protocols&#34; : [ &#34;https&#34; ],
      &#34;signatureVersions&#34; : [ &#34;v4&#34; ],
      &#34;variants&#34; : [ {
        &#34;dnsSuffix&#34; : &#34;amazonaws.com&#34;,
        &#34;hostname&#34; : &#34;{service}-fips.{region}.{dnsSuffix}&#34;,
        &#34;tags&#34; : [ &#34;fips&#34; ]
      }, {
        &#34;dnsSuffix&#34; : &#34;api.aws&#34;,
        &#34;hostname&#34; : &#34;{service}-fips.{region}.{dnsSuffix}&#34;,
        &#34;tags&#34; : [ &#34;dualstack&#34;, &#34;fips&#34; ]
      }, {
        &#34;dnsSuffix&#34; : &#34;api.aws&#34;,
        &#34;hostname&#34; : &#34;{service}.{region}.{dnsSuffix}&#34;,
        &#34;tags&#34; : [ &#34;dualstack&#34; ]
      } ]
    },
    &#34;dnsSuffix&#34; : &#34;amazonaws.com&#34;,
    &#34;partition&#34; : &#34;aws-us-gov&#34;,
    &#34;partitionName&#34; : &#34;AWS GovCloud (US)&#34;,
    &#34;regionRegex&#34; : &#34;^us\\-gov\\-\\w&#43;\\-\\d&#43;$&#34;,
    &#34;regions&#34; : {
      &#34;us-gov-east-1&#34; : {
        &#34;description&#34; : &#34;AWS GovCloud (US-East)&#34;
      },
      &#34;us-gov-west-1&#34; : {
        &#34;description&#34; : &#34;AWS GovCloud (US-West)&#34;
      }
```

#### Регионы
```go
const (
    UsGovEast1RegionID = &#34;us-gov-east-1&#34; // AWS GovCloud (US-East).
    UsGovWest1RegionID = &#34;us-gov-west-1&#34; // AWS GovCloud (US-West).
)
```

## Secret Partitions
Секретные регионы AWS отличаются от GovCloud и представляет собой регион, созданный Разведывательным сообществом США (IC) как для своих нужд, так и для предоставления коммерческих облачных услуг (C2S).  

ISO partition находятся в домене ic.gov и мы можем проверить, кому принадлежит он принадлежит. На [Whois сайте для доменов .gov](https://domains.dotgov.gov/dotgov-web/registration/whois.xhtml) мы обнаружим, домен принадлежит Central Intelligence Agency, оно же ЦРУ.  

Секретность тут довольно спорная. Указано, что данные разделы используются IC, CIA и прочими государственными органами США, однако часть информации можно о них узнать в [Go SDK](https://docs.aws.amazon.com/sdk-for-go/api/aws/endpoints/), и в issue [terraform-провайдера](https://github.com/hashicorp/terraform-provider-aws/issues/18593).

Так же список эндпоинтов указаны в [botocore](), [aws-sdk-js-v3](https://github.com/aws/aws-sdk-js-v3/blob/c96f35f972c44706a391bb07e0a897e73b8d6dfe/clients/client-cloudfront/endpoints.ts) и некоторых других инструментах.

В данном случае я буду рассматривать [endpoints.json]() из репозитория botocore.

### Partition AWS ISO (США)
| AWS Partition code | AWS Partition Name        |
| ------------------ | ------------------------- |
| aws-iso            | AWS ISO (US) - Top Secret |

__AWS-iso__ был запущен для обслуживания Top secret government data.
На данный момент есть два региона, которые были развернуты в рамках разных контрактов: 
1. US ISO East был запущен в 2014 
2. US ISO West был запущен в 2021.  

Он не является частью общедоступного Интернета. Заявленно, что присуствует air-gap, но описание сервисов доступно в конфигурации инструментов.  
Создан и запущен силами AWS, но размещается в ЦРУ (on-premise).  

DNS-суффикс: `c2s.ic.gov`

#### Общее описание из endpoints
```json
{
    &#34;defaults&#34; : {
      &#34;hostname&#34; : &#34;{service}.{region}.{dnsSuffix}&#34;,
      &#34;protocols&#34; : [ &#34;https&#34; ],
      &#34;signatureVersions&#34; : [ &#34;v4&#34; ],
      &#34;variants&#34; : [ {
        &#34;dnsSuffix&#34; : &#34;c2s.ic.gov&#34;,
        &#34;hostname&#34; : &#34;{service}-fips.{region}.{dnsSuffix}&#34;,
        &#34;tags&#34; : [ &#34;fips&#34; ]
      } ]
    },
    &#34;dnsSuffix&#34; : &#34;c2s.ic.gov&#34;,
    &#34;partition&#34; : &#34;aws-iso&#34;,
    &#34;partitionName&#34; : &#34;AWS ISO (US)&#34;,
    &#34;regionRegex&#34; : &#34;^us\\-iso\\-\\w&#43;\\-\\d&#43;$&#34;,
    &#34;regions&#34; : {
      &#34;us-iso-east-1&#34; : {
        &#34;description&#34; : &#34;US ISO East&#34;
      },
      &#34;us-iso-west-1&#34; : {
        &#34;description&#34; : &#34;US ISO WEST&#34;
      }
```

#### Регионы
```go
const (
    UsIsoEast1RegionID = &#34;us-iso-east-1&#34; // US ISO East.
    UsIsoWest1RegionID = &#34;us-iso-west-1&#34; // US ISO WEST.
)
```

#### IAM
```yaml
{
&#34;iam&#34; : {
        &#34;endpoints&#34; : {
          &#34;aws-iso-global&#34; : {
            &#34;credentialScope&#34; : {
              &#34;region&#34; : &#34;us-iso-east-1&#34;
            },
            &#34;hostname&#34; : &#34;iam.us-iso-east-1.c2s.ic.gov&#34;
          }
        },
        &#34;isRegionalized&#34; : false,
        &#34;partitionEndpoint&#34; : &#34;aws-iso-global&#34;
      },
```

### Partition AWS ISO B (США)
| AWS Partition code | AWS Partition Name                     |
| ------------------ | -------------------------------------- |
| aws-iso-b          | AWS ISOB (US) - Secret government data |

__aws-iso-b__ был запущен в 2017.  
Может работать с рабочими нагрузками вплоть до секретного уровня.  
aws-iso-b доступен как IC, так и сторонним гос. клиентам, и может использоваться в любых государственных учреждениях.  

DNS-суффикс: `sc2s.sgov.gov`

#### Общее описание из endpoints
```json
{
    &#34;defaults&#34; : {
      &#34;hostname&#34; : &#34;{service}.{region}.{dnsSuffix}&#34;,
      &#34;protocols&#34; : [ &#34;https&#34; ],
      &#34;signatureVersions&#34; : [ &#34;v4&#34; ],
      &#34;variants&#34; : [ {
        &#34;dnsSuffix&#34; : &#34;sc2s.sgov.gov&#34;,
        &#34;hostname&#34; : &#34;{service}-fips.{region}.{dnsSuffix}&#34;,
        &#34;tags&#34; : [ &#34;fips&#34; ]
      } ]
    },
    &#34;dnsSuffix&#34; : &#34;sc2s.sgov.gov&#34;,
    &#34;partition&#34; : &#34;aws-iso-b&#34;,
    &#34;partitionName&#34; : &#34;AWS ISOB (US)&#34;,
    &#34;regionRegex&#34; : &#34;^us\\-isob\\-\\w&#43;\\-\\d&#43;$&#34;,
    &#34;regions&#34; : {
      &#34;us-isob-east-1&#34; : {
        &#34;description&#34; : &#34;US ISOB East (Ohio)&#34;
      }
```
#### IAM
```json
{
      &#34;iam&#34; : {
        &#34;endpoints&#34; : {
          &#34;aws-iso-b-global&#34; : {
            &#34;credentialScope&#34; : {
              &#34;region&#34; : &#34;us-isob-east-1&#34;
            },
            &#34;hostname&#34; : &#34;iam.us-isob-east-1.sc2s.sgov.gov&#34;
          }
        },
        &#34;isRegionalized&#34; : false,
        &#34;partitionEndpoint&#34; : &#34;aws-iso-b-global&#34;
      },
```

#### Регионы
```json
{
  &#34;aws-iso-b&#34;,
  &#34;AWS ISOB (US)&#34;,
  }{
    &#34;us-isob-east-1&#34;: {
      &#34;description&#34;: &#34;US ISOB East (Ohio)&#34;
    }
  }
```

### Partition AWS ISO E (Европа)
| AWS Partition code | AWS Partition Name        |
| ------------------ | ------------------------- |
| aws-iso-e          | AWS ISOE (Eu) - ?         |

Не известна дата запуска, известно только то, что оно сделано для Европы.   
Вероятно, этот partition создаётся так же для правительственных нужд или нужд британской разведки.  

Известен DNS-суффикс: `cloud.adc-e.uk`.  

#### Регионы
Нет указанных регионов, но судя по суффиксу, будет размещаться в Великобритании.

#### Общее описание из endpoints
```json
   {
    &#34;defaults&#34; : {
      &#34;hostname&#34; : &#34;{service}.{region}.{dnsSuffix}&#34;,
      &#34;protocols&#34; : [ &#34;https&#34; ],
      &#34;signatureVersions&#34; : [ &#34;v4&#34; ],
      &#34;variants&#34; : [ {
        &#34;dnsSuffix&#34; : &#34;cloud.adc-e.uk&#34;,
        &#34;hostname&#34; : &#34;{service}-fips.{region}.{dnsSuffix}&#34;,
        &#34;tags&#34; : [ &#34;fips&#34; ]
      } ]
    },
    &#34;dnsSuffix&#34; : &#34;cloud.adc-e.uk&#34;,
    &#34;partition&#34; : &#34;aws-iso-e&#34;,
    &#34;partitionName&#34; : &#34;AWS ISOE (Europe)&#34;,
    &#34;regionRegex&#34; : &#34;^eu\\-isoe\\-\\w&#43;\\-\\d&#43;$&#34;,
    &#34;regions&#34; : { },
    &#34;services&#34; : { }
  } 

```

### Partition AWS ISO F
| AWS Partition code | AWS Partition Name        |
| ------------------ | ------------------------- |
| aws-iso-f          | AWS ISOF - ?              |

Не удалось найти что-либо, возможно ещё не дошло до стадии строительства.  

Известен только DNS-суффикс: `csp.hci.ic.gov`.  

#### Регионы
Нет указанных регионов.

#### Общее описание из endpoints
```json
  {
    &#34;defaults&#34; : {
      &#34;hostname&#34; : &#34;{service}.{region}.{dnsSuffix}&#34;,
      &#34;variants&#34; : [ {
        &#34;dnsSuffix&#34; : &#34;csp.hci.ic.gov&#34;,
        &#34;hostname&#34; : &#34;{service}-fips.{region}.{dnsSuffix}&#34;,
        &#34;tags&#34; : [ &#34;fips&#34; ]
      } ]
    },
    &#34;dnsSuffix&#34; : &#34;csp.hci.ic.gov&#34;,
    &#34;partition&#34; : &#34;aws-iso-f&#34;,
    &#34;partitionName&#34; : &#34;AWS ISOF&#34;,
    &#34;regionRegex&#34; : &#34;^us\\-isof\\-\\w&#43;\\-\\d&#43;$&#34;,
    &#34;regions&#34; : { },
    &#34;services&#34; : { }
  }
```
Есть ли какие-то ещё разделы и регионы не известно, в официальных источниках указаны только коммерческие регионы и разделы.

### Финансовый вопрос
Информации о сделках не так много. Если рассматривать только контракты с правительством США, то были контракты на \\$600 миллионов в 2013-2014 годах (судя по всему, это за aws-iso), в 2021 был заключен контракт уже на $10 миллиардов. Возможно, цифра не впечатляющая, однако это только часть пазла. Так же [некоторые источники сообщают](https://www.tenderalpha.com/blog/post/quantitative-analysis/how-much-do-government-contracts-contribute-to-amazon-web-services-growth), что AWS получила государственный контракт, потенциальная сумма которого к 2026 году может достичь $178,8 млн.

## Резюме
Несколько странно, что такая информация об этом попадает вообще в публичный доступ, в том числе SDK.  
Само собой, это нельзя назвать какой-то утечкой информации или поводом для выезда пати-вэна к техническим писателям и разработчикам.  
  
Но это позволяет посмотреть на AWS несколько под иным углом, даёт представление о том, где американские рыцари плаща и кинжала держат свою инфраструктуру.  


---
## Reference
### AWS
[AWS Docs: Partitions](https://docs.aws.amazon.com/whitepapers/latest/aws-fault-isolation-boundaries/partitions.html)  
[AWS GovCloud](https://aws.amazon.com/govcloud-us/)  
[AWS: AWS GovCloud (US) or standard? Selecting the right AWS partition](https://aws.amazon.com/blogs/publicsector/aws-govcloud-us-standard-selecting-right-aws-partition/)  
[AWS: Announcing the New AWS Secret Region](https://aws.amazon.com/blogs/publicsector/announcing-the-new-aws-secret-region/)  
[AWS Docs: AWS SDK for Go API Reference. pkg constants](https://docs.aws.amazon.com/sdk-for-go/api/aws/endpoints/#pkg-constants)  
[AWS Docs: Amazon Resource Names (ARNs)](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html)  
### AWS Public Sector Summit
[Youtube: AWS Public Sector Summit 2017 Customer Keynote: John Edwards, Central Intelligence Agency](https://www.youtube.com/watch?v=DEc6kVAXSs8)
### Blogs
[Neokobo: AWS Regions in Order by Partition Type](https://neokobo.blogspot.com/2022/02/aws-regions-in-order-by-partition-type.html)  
[note.com: 知らなくても困らないAWSリージョンのパーティション](https://note.com/lighthawk/n/n13cef41d978b)  
### Github
[Github botocore. endpoints]
[Github: Hashicorp Issue: Support aws-iso, aws-iso-b, aws-iso-e, aws-iso-f partitions](https://github.com/hashicorp/terraform-provider-aws/issues/18593)  
[Github: prowler. aws_regions_by_service](https://github.com/prowler-cloud/prowler/blob/master/prowler/providers/aws/aws_regions_by_service.json)  
[Github: aws-sdk-go. endpoints](https://github.com/aws/aws-sdk-go/blob/main/models/endpoints/endpoints.json)  
[Github: aws-sdk-js-v3. endpoints](https://github.com/aws/aws-sdk-js-v3/blob/c96f35f972c44706a391bb07e0a897e73b8d6dfe/clients/client-cloudfront/endpoints.ts)  
### Finance
[The Atlantic: The Details About the CIA&#39;s Deal With Amazon](https://www.theatlantic.com/technology/archive/2014/07/the-details-about-the-cias-deal-with-amazon/374632/)  
[Nextgov: Amazon Web Services Announces Second ‘Top Secret’ Cloud Region](https://www.nextgov.com/emerging-tech/2021/12/amazon-web-services-announces-second-top-secret-cloud-region/187303/)  
[Tender Alpha: How Much Do Government Contracts Contribute to Amazon Web Services’ Growth?](https://www.tenderalpha.com/blog/post/quantitative-analysis/how-much-do-government-contracts-contribute-to-amazon-web-services-growth)
### Other
[Wiki: United States Intelligence Community](https://en.wikipedia.org/wiki/United_States_Intelligence_Community)  
[AWSRegion.info](https://awsregion.info/)  

---

> Author:   
> URL: http://localhost:1313/posts/cloud_jackets_aws_for_cia/  

