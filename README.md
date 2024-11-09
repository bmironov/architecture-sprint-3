# Базовая настройка

## Запуск minikube

[Инструкция по установке](https://minikube.sigs.k8s.io/docs/start/)

```bash
minikube start
```

## Добавление токена авторизации GitHub

[Получение токена](https://github.com/settings/tokens/new)

```bash
kubectl create secret docker-registry ghcr --docker-server=https://ghcr.io --docker-username=<github_username> --docker-password=<github_token> -n default
```

## Установка API GW kusk

[Install Kusk CLI](https://docs.kusk.io/getting-started/install-kusk-cli)

```bash
kusk cluster install
```

## Смена адреса образа в helm chart

После того как вы сделали форк репозитория и у вас в репозитории отработал GitHub Action. Вам нужно получить адрес образа <https://github.com/><github_username>/architecture-sprint-3/pkgs/container/architecture-sprint-3

Он выглядит таким образом
```ghcr.io/<github_username>/architecture-sprint-3:latest```

Замените адрес образа в файле `helm/smart-home-monolith/values.yaml` на полученный файл:

```yaml
image:
  repository: ghcr.io/<github_username>/architecture-sprint-3
  tag: latest
```

## Настройка terraform

[Установите Terraform](https://yandex.cloud/ru/docs/tutorials/infrastructure-management/terraform-quickstart#install-terraform)

Создайте файл ~/.terraformrc

```hcl
provider_installation {
  network_mirror {
    url = "https://terraform-mirror.yandexcloud.net/"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
```

## Применяем terraform конфигурацию

```bash
cd terraform
terraform init
terraform apply
```

## Настройка API GW

```bash
kusk deploy -i api.yaml
```

## Проверяем работоспособность

```bash
kubectl port-forward svc/kusk-gateway-envoy-fleet -n kusk-system 8080:80
curl localhost:8080/hello
```

## Delete minikube

```bash
minikube delete
```


## Задание 1

### Общий анализ приложения As-Is

Насколько я понимаю Java и не имею никакого представления о Spring - в этом
приложении слишком мало кода. В нем скорее слишком много недостающего функционала,
чем чего-то полезного для бизнеса.

Например, хоть и объявлены структуры данных для температурных сенсоров, в приложении
начисто отсутствуют методы коммуникации чтобы управлять этими сущностями.
Так же вызывает вопрос и решение об отсутствии интерфейса для наполнения таблицы
heating_systems данными. А без данных в этой таблице все попытки что-то изменить
будут заканчиваться печально. Видимо этой деятельностью занимаются люди в офисе
с прямым доступом к БД.

Если принять, что система отопления имеет внутренний цикл около минуты, то при
100 подключенных устройствах к нашему монолиту в среднем в синхронном режиме 
надо иметь время отклика монолита около 0.6 секунды. Это вполне приемлимая
скорость работы и тут вряд ли какой-нибудь из компонентов станет узким местом.

Масштабировать такой монолит можно, но в ограниченных объемах. Так как все
коммуникации в приложении stateless, то вполне возможно запустить несколько
экземпляров приложения и вместо ```/src/main/resources/application.yml```
указывать порт через параметр JVM как ```jvm -Dserver.port=8090```. Тогда на одном
хосте при наличии ЦПУ и ОЗУ можно обрабатывать в два раза больше запросов при
двух экземплярах приложения. Балансировку между ними можно запросто делать с
помощью HAProxy. При достаточном количесстве экземрляров можно сгладить достаточно
большое время на запуск и перезапуск экземпляра JVM. Так же это позволит использовать
и методы "канарейки" или "green-blue" для постепенного обновления приложения.

Масштабировать на уровне БД вряд ли придется так как Primary Key constraint
уже есть у каждой из таблиц. Пока там не будет миллионов записей большого смысла в их
партиционировании при работе только через PK индекс не будет. Размер записей
в этих таблицах тоже мал, поэтому и файлы данных и индексов будут сравнительно
небольшими. Тут PostgreSQL по своему обычаю будет полагаться и выигрывать от кэша 
ОС, которая будет с удовольствием хранить их в свем кэше.

Для повышения надежности уровня БД необходимо иметь 2 копии БД (primary и standby).
Это автоматически позволяет добавть возможность разделения пишущего и читающего
трафика в БД с помощью того же HAProxy. При большом количестве устройств, скорее
всего пишущая нагрузка будет превышать читающую и тут PostgreSQL в конфигурации
физической реплики станет узким местом. Далее это можно будет преодолеть с
помощью переключение на логическую репликацию в master-master конфигурации. Тут,
конечно же, всплывут проблемы обновления данных на одном сервере БД и дальнейшая
попытка прочитать эти же данные, но уже с другого сервера до окончания репликации
между ними. В данном приложении эту проблему можно решить через кэш.

И "последним вздохом имератора" (нашего монолита) может стать попытка ускорить
слой БД за счет добавления Redis. Тогда запросы типа GET ```/{id}``` и
```/{id}/current-temperature``` можно обслуживать быстрее, а информацию в Redis
обновлять во время выполнения соответствующих PUT и POST запросов.

Кстати, что интересно - это то, что система в принципе не обрабатывает DELETE
запросы. Это означает, что ошибочно введенные данные (например, новые устройства)
не имеют возможности быть удалены. Но, видимо, этим тоже занимаются люди в офисе
с прямым доступом к БД.

В итоге можно сказать, что у данного приложения есть перспективы по масштабируемости
для обслуживания более чем текущие 100 устройств раз в 10. Но все же, если в
планах заняться обслуживанием 100 * 200 * 5 = 100'000 устройств, то тут монолиту
будет тяжело. Ведь при тех же ежеминутных обновлениях это уже 0.0006 сек для
синхронной обработки данных в один канал.

### Домены и границы контекстов

Существующая система состоит из нескольких доменов. Кое-какие из них видны в
исходном коде приложения-монолита, остальные необходимо додумывать, так как без
них вряд ли может существовать более-менее жизнеспособное приложение.

#### Домены

К основным доменам в компании "Тёплый дом" можно отнести:
- домен управления информацией о клиентах;
- домен управления системами отопления и их телеметрией.

Так же должен быть представлены:
- домен клиента и все устройства в его доме;
- домен внешних приложений, которые должны управлять умными устройствами и
взаимодействовать с системами компании "Тёплый дом".

#### Контекст

Контекстно домены делятся на свои составляющие.
Домен управления информацией о клиентах имеет следующие контексты:
- управление реестром пользователей (поддержание списка текущих клиентов компании);
- управление сессиями пользоватедей.

Домен управления системами отопления и их телеметрии предлагается разбить на
сделующие контексты:
- управление реестром сиситем отопления (поддержание списка текущих систем);
- управление системами отопления (отправка сигналов системам отопления);
- управление телеметрией систем отопления (сохранять и предоставлять по запросу).

### Domain-Driven-Design

Я бы предложил делить монолит на домены по типу устройств. Закон Конвея нам тут
не мешает, так как компания достаточно маленькая. А вот задав общий метод как
"один тип устройств = один микросервис" мы получаем гибкость как во внедрении
нового типа устройств, когда не надо будет открыватели ворот управлять теми же
интерфейсами, что и лампочки в доме. Так и снижение риска при таком внедрении,
когда новый функционал никак не влияет на инфраструктуру остальных доменов.

У каждого из таких доменов будут четкие контексты, которые присущи именно
конкретному типу устройств. Например, не только "Включить", "Выключить" и "Снять
показания", но и "Открыть", "Закрыть" и т.п.

Так как разработка приложения пользователя отдана другому коллемктиву, то
во внутренней инфраструктуре необходимо подчеркнуть еще один домен - авторизации.
В приложении подобный функционал отсутствует, но для сторонних приложений
необходимо иметь какой-то функционал в этой области. Так как компания должна
хранить информацию о своих клиентах, то такое хранилище - скорее всего БД и из-за
размера коллектива компании, скорее всего - это тот же PostgreSQL. На жиаграмме
эта инфраструктура отнесена в область "Остальная инфраструктура компании".

В итоге, с предложенным DDD "один тип устройств = один микросервис" мы получаем
как минимум N+1 микросервисов в To-Be дизайне (N типов устройств и плюс 1 для
авторизации). На данном этапе я не уверен, что есть необходимость выделения из
всех этих микросервисов функционала телеметрии в отдельный микросервис. В этом
вопросе есть веские "за", и "против". Например, сбор телеметрии любых устройств
из топиков шины всего одним сервисом и отсутствие тем самым дублирования кода
между микросервисами. Против превращения БД такого микросервиса в гигантскую
свалку данных разного формата (тут я не имею в виду "JSON все переварит"), с
которыми вряд ли можно будет делать что-то полезное. Так что - вопрос не
однозначен.

### Диаграммы

Задание по построению диаграм разбито на 3 части:
1. Диаграмма [доменов](./diagrams/task_1_domains.puml).
2. Диаграмма [контекстов](./diagrams/task_1_context.puml).
3. Диаграмма [монолита](./diagrams/task_1_monolith.puml).

## Задание 2

Решение о делении монолита на микросервисы по типу устройст имеет очень большое
преимущество над другими методами в плане риска. Ведь в итоге мы получаем, что
текущий монолит остается в своем изначальном виде и продолжает работать. Все новые
типы устройств для включения их в бизнес будут требовать создания своего
индивидуального микросервиса. А когда придет время дробить текущий монолит систем
отопления, то можно будет использовать один из подходящих методов.

Система авторизации в репозитории не представлена. Скорее всего её так же
придется преобразовать в микросервис хотя бы для возможности масштабирования в
связи с дополнительной нагрузкой от новых микросервисов и на порядки большего
числа клиентов.

### Диаграммма контейнеров

На [диаграмме контейнеров](./diagrams/task_2_containers.puml) показаны текущий
монолит систем отопления и еще 3 микросервиса других типо устройств. Полагаю
основной принцип добавления новых типов устройств понятен и смысла добавлять
еще микросервисы нет, так как автоматически сгенерированная диаграмма уже слабо
читаема.

### Диаграммма компонентов

Так как было принято решение дробить монолит на микросервисы по типу устройств,
то на [диаграмме компонентов](./diagrams/task_2_components.puml) показаны для
примера только 2 таких микросервиса. Остальные 3 микросервиса будут идентичны по
внутренней структуре, а вот наглядность диаграммы уже с 2-мя сервисами начинает
терять простоту и изящность. 

Еще хочу добавить, что не вижу смысла делать в каждом микросервисе свой персональный
процесс поддержания очереди (я выбрал Kafka), так как сама по себе Kafka хорошо
масштабируется для нагрузки. Считаю, что это будет перерасходом ресурсов делать
кластера Kafka в каждом из микросервисов. Но если это потребуется, то достаточно
легко можно будет внедрить персональную Kafka в каком-то выделенном микросервисе.
Я работал в больших компаниях и там выбирался именно такой подход, когда один
кластер Kafka обслуживал всю корпорацию.

### Диаграммма кода

Так как приложение создано с помощью фрэймворка, то построить его поноценную
[диаграмму кода](./diagrams/task_2_code.puml) довольно затруднительно. Это
связано с тем, что мы пишем маленькие кусочки кода, которые встраиваются в большое
количество кода самого фрймворка. Поэтому и даиграмма у меня получилась скучной
и мало что показывающей. Поэтому я попытался восполнить этот недостаток
[другой диаграммой](./diagrams/task_2_sequence.puml)

## Задание 3

[ER диаграмма](./diagrams/task_3_er.puml) содержит основные таблицы для проекта
уровня MVP. Не совсем понятно зачем нужен Module. Поэтому делаем предположение,
что Module - это какой-то модуль, который добавляет функционал Device (и таких
модулей может быть несколько). Например, Алиса-колонка - это Device, а TV,
светильник или чайник - это модули для этой Алисы. Их этого делаем вывод, что
телеметрию могут поставлять как Device, так и все его модули.

На правах составителя диаграммы в абсолютно новом проекте предпочитаю ввести
практику именования сущностей не изменением регистра (это удобно в Java), а через
подчерк - это удобно при работе с БД. Обычно бывает необходимость что-то делать
с таблицами на уровне SQL и тогда это мучение выискивать нужную таблицу, где
все они имеют имена длинной под 60 символов, созданы как одно слово и отличаются
где-то в средних буквах. При этом БД переводит такое "camelCase" имя в один
регистр (верхний или нижний). Такое же мучение и с написанием каких-то ad-hoc SQL
запросов к БД пока они не реализованы в приложении (например, в помощь отделу
поддержки).

Это так же совпадает с очень хорошей практикой представленной в Ruby On Rails,
где имя таблицы писалось через подчерки и суррогатный PK в таблице именовался как
<table_name>_id. В этом случае все связки в условиях WHERE не требуют никаких
дополнительных адиасов чтобы отличить колонку ID одной таблицы от другой.

## Задание 4

[OpenAPI](./diagrams/task_3.1_openapi.puml) диаграмма построена на основе
предлагаемого по умолчанию варианта API для зоомагазина "Pet Store".

Выделены 3 большие группы интерфейсов для:
- работы с БД клиентов (```/users```)
- поддержки систем отопления (```/hvacs```) [HVAC = Heating, Ventilation, Air Conditioning]
- поддержки систем освещения (```/lights```)

HVAC endpoint поддерживает основной функционал управления записями о системах
отопления (создание, обновление, выборку и удаление) в домах клиентов компании.
Так же есть возможность для сбора телеметрии (```/hvacs/{hvacID}/telemetry```)
с каждого устройства, и отправки управляющего сигнала
(```/hvacs/{hvacID}/signal```) конкретной системе отопления.
Оставим за скобками протокол того, как происходит непосредственная отправка
управляющих сигналов такому устройству и то, как телеметрия с него
маршрутизируется на endpoiont нашего микросервиса.

Системы освещения обрабатываются схожим образом с системами отопления с небольшой
разницей в типе телеметрии (процент яркости вместо температуры).

Остальные умные устройства предполагается обслуживать по схожей схеме с
```/hvacs``` и ```/lights```.