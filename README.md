# Running-Airflow-in-Docker

### Это краткое руководство позволит вам быстро запустить Airflow с помощью CeleryExecutor в Docker
`Эта процедура может быть полезна для обучения и исследования. Однако адаптация его для использования в реальных ситуациях может быть сложной, и файл компоновки Docker не предоставляет никаких гарантий безопасности, необходимых для производственной системы. Для внесения изменений в эту процедуру потребуются специальные знания в Docker и Docker Compose`
### По этой причине мы рекомендуем использовать Kubernetes с официальной диаграммой управления сообщества Airflow, когда вы будете готовы запустить Airflow в рабочей среде
## Прежде чем начать
Эта процедура предполагает знакомство с Docker и Docker Compose. Если вы раньше не работали с этими инструментами, вам следует потратить немного времени на изучение Docker Quick Start (особенно раздел Docker Compose), чтобы вы знали, как они работают.

Выполните следующие действия, чтобы установить необходимые инструменты, если вы еще этого не сделали.

    1. Установите Docker Community Edition (CE) на свою рабочую станцию. В зависимости от вашей ОС вам может потребоваться настроить Docker на использование не менее 4,00 ГБ памяти для правильной работы контейнеров Airflow. Дополнительную информацию можно найти в разделе «Ресурсы» документации Docker для Windows или Docker для Mac.

    2. Установите Docker Compose v2.14.0 или новее на свою рабочую станцию.

Старые версии docker-compose не поддерживают все функции, необходимые для файла docker-compose.yaml Airflow, поэтому дважды проверьте, соответствует ли ваша версия минимальным требованиям к версии.

Объема памяти по умолчанию, доступного для Docker в macOS, часто недостаточно для запуска Airflow. Если недостаточно памяти выделено, это может привести к постоянному перезапуску веб-сервера. Вам следует выделить как минимум 4 ГБ памяти для Docker Engine (в идеале 8 ГБ).

Вы можете проверить, достаточно ли у вас памяти, выполнив эту команду:

`docker run --rm "debian:bookworm-slim" bash -c 'numfmt --to iec $(echo $(($(getconf _PHYS_PAGES) * $(getconf PAGE_SIZE))))'`

## Получение docker-compose.yaml

Для развертывания Airflow на Docker Compose, вы должны получить docker-compose.yaml.

`curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.10.5/docker-compose.yaml'`

С июля 2023 года Compose V1 перестал получать обновления. Рекомендуется обновлять Docker Compose на более новую версию, поскольку docker-compose.yaml может не работать в Compose V1.

Этот файл содержит несколько определений служб:

    airflow-scheduler - Планировщик контролирует все задачи и DAG, а затем запускает экземпляры задач после завершения их зависимостей.

    airflow-webserver - Веб-сервер доступен по адресу http://localhost:8080.

    airflow-worker - Рабочий, выполняющий задачи, указанные планировщиком.

    airflow-triggerer - Triggerer запускает цикл событий для переносимых задач.

    airflow-init - Служба инициализации.

    postgres - База данных.

    redis - Переадресатор, который передает сообщения от планировщика к работнику.

Вы можете включить flower, добавив опцию --profile flower, например docker compose --profile flower up, или прямо указав её на командной строке, например docker compose up flower.

    flower - Приложение для мониторинга среды. Доступно на http://localhost:5555.

Все эти службы позволяют выполнять Airflow с помощью CeleryExecutor. Более подробную информацию см. в разделе Обзор архитектуры.

Некоторые каталоги в контейнере монтированы, что означает, что их содержимое синхронизировано между вашим компьютером и контейнером.

    . /dags - можете положить свои файлы DAG здесь.

    . /logs - содержит журналы выполнения заданий и планировщика.

    . /config - можете добавить пользовательский логотип-анализатор или airflow_local_settings.py для настройки политики кластера.

    . /plugins - здесь вы можете добавить свои собственные plugins.

Этот файл использует последний образ Airflow (apache/airflow). Если вам нужно установить новую библиотеку или системную библиотеку Python, вы можете создать свой образ.

## Инициализация Environment

Перед первым запуском Airflow необходимо подготовить среду, т.е. создать необходимые файлы, каталоги и инициализировать базу данных.

### Установка пользователя Airflow

На Linux, быстрый запуск должен знать свой хост-идентификатор пользователя и должен иметь групповой идентификатор установлен на 0. В противном случае файлы, созданные в дагах, журналах и плагинов будут созданы с правами корневого пользователя. Вы должны убедиться, что они настроены для docker-compose:

`mkdir -p ./dags ./logs ./plugins ./config`

`echo -e "AIRFLOW_UID=$(id -u)" > .env`

Смотрите [переменные среды Docker Compose](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#docker-compose-env-variables)

Для других операционных систем может появиться предупреждение, что AIRFLOW_UID не установлен, но вы можете спокойно проигнорировать его. Также можно вручную создать .env файл в той же папке, что docker-compose.yaml с этим содержанием, чтобы избавиться от предупреждения:

`AIRFLOW_UID=50000`

### Инициализировать базу данных

На всех операционных системах необходимо выполнить миграцию баз данных и создать первую учетную запись пользователя. Для этого запустите:

`docker compose up airflow-init`

После завершения инициализации вы должны увидеть следующее сообщение:

`airflow-init_1       | Upgrades done`

`airflow-init_1       | Admin user airflow created`

`airflow-init_1       | 2.10.5`

`start_airflow-init_1 exited with code 0`

Созданная учетная запись имеет имя пользователя и пароль airflow.

## Очистка Environment

Разработанная нами среда docker-compose не был разработан для использования в производстве и имеет ряд оговорок - один из них, что лучший способ восстановить от любой проблемы, чтобы очистить его и перезапустить с нуля:

    Запустить команду `docker compose down --volumes --remove-orphans` в каталоге, где вы загрузили docker-compose.yaml файл

    Удалить всю папку, в которой вы загрузили docker-compose.yaml файл `rm -rf '<DIRECTORY>'`

    Пройдите по этому руководству с самого начала, начиная с перезагрузки docker-compose.yaml

## Запуск Airflow

Теперь вы можете запустить все службы:

`docker compose up`

Во втором терминале можно проверить состояние контейнеров и убедиться, что ни один из них не находится в нерабочем состоянии:

`$ docker ps`
|`CONTAINER ID`      |`IMAGE`             |`COMMAND`            |`CREATED`         |`STATUS`          |`PORTS`                | `NAMES`                         |
|:-------------------|:-------------------|:--------------------|:-----------------|:-----------------|:----------------------|:--------------------------------|
|`247ebe6cf87a`|`apache/airflow:2.10.5`|`"/usr/bin/dumb-init …"`|`3 minutes ago`|`Up 3 minutes (healthy)`|`8080/tcp`|`compose_airflow-worker_1`|
|`ed9b09fc84b1`|`apache/airflow:2.10.5`|`"/usr/bin/dumb-init …"`|`3 minutes ago`|`Up 3 minutes (healthy)`|`8080/tcp`|`compose_airflow-scheduler_1`|
|`7cb1fb603a98`|`apache/airflow:2.10.5`|`"/usr/bin/dumb-init …"`|`3 minutes ago`|`Up 3 minutes (healthy)`|`0.0.0.0:8080->8080/tcp`|`compose_airflow-webserver_1`|
|`74f3bbe506eb`|`postgres:13`|`"docker-entrypoint.s…"`|`18 minutes ago`|`Up 17 minutes (healthy)`|`5432/tcp`|`compose_postgres_1`|
|`0bd6576d23cb`|`redis:latest`|`"docker-entrypoint.s…"`|`10 hours ago`|`Up 17 minutes (healthy)`|`0.0.0.0:6379->6379/tcp`|`compose_redis_1`|

## Доступ к Environment

После запуска Airflow вы можете взаимодействовать с ним 3-мя способами:

    выполняя команды CLI.

    через браузер, используя веб-интерфейс.

    с помощью REST API.

### Взаимодействие через команды CLI

Вы также можете выполнять команды CLI, но это нужно делать в одном из определенных служб airflow-*. Например, чтобы запустить airflow info, запустите следующую команду:

`docker compose run airflow-worker airflow info`

Если у вас Linux или Mac OS, вы можете сделать вашу работу легче и загрузить дополнительные скрипты wrapper, которые позволят вам выполнять команды с более простой командой.

`curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.10.5/airflow.sh'
chmod +x airflow.sh`

Теперь вы можете выполнять команды проще.

`./airflow.sh info`

Вы также можете использовать bash как параметр для ввода интерактивной оболочки bash в контейнер или python для ввода контейнера python.

`./airflow.sh bash`

`./airflow.sh python`

### Взаимодействие через веб-интерфейс

После запуска кластера вы можете войти в веб-интерфейс и начать эксперименты с DAG.

Веб-сервер доступен по адресу: `http://localhost:8080`. Учетная запись по умолчанию имеет имя и пароль `airflow`.

### Взаимодействие через REST API

[Базовая аутентификация](https://en.wikipedia.org/wiki/Basic_access_authentication) пароля пользователя в настоящее время поддерживается для REST API, что означает, что вы можете использовать общие инструменты для отправки запросов к API.

Веб-сервер доступен по адресу: `http://localhost:8080`. Учетная запись по умолчанию имеет имя и пароль `airflow`.

Вот пример команды `curl`, которая отправляет запрос на получение списка пулов:

`ENDPOINT_URL="http://localhost:8080/"
 curl -X GET  \
     --user "airflow:airflow" \
     "${ENDPOINT_URL}/api/v1/pools"`

## Удаление (очистка)

Для остановки и удаления контейнеров, удаления томов с данными базы данных и загрузки изображений, выполните:

`docker compose down --volumes --rmi all`

## Использование пользовательских образов (images)

Когда вы хотите запустить Airflow локально, вы можете использовать расширенный образ, содержащий некоторые дополнительные зависимости - например, вы можете добавить новые пакеты python или обновить провайдеры airflow на более позднюю версию. Это можно сделать очень легко, указав `build: .` в вашем `docker-compose.yaml` и поместив настраиваемый докерфайл рядом с вашим `docker-compose.yaml`. Затем вы можете использовать команду `docker compose build` для создания образа (вам нужно сделать это только один раз). Вы также можете добавить флаг `--build` к вашим командам создания докера, чтобы воссоздавать изображения "на лету", когда вы запускаете другие команды создания докера.

Примеры того, как вы можете расширить образ с помощью пользовательских провайдеров, пакетов python, пакетов apt и других, можно найти в разделе [Создание образа](https://airflow.apache.org/docs/docker-stack/build.html).

`ПРИМЕЧАНИЕ: Создание пользовательских изображений означает, что вам нужно поддерживать также уровень автоматизации, так как вам необходимо воссоздавать изображения, когда вы хотите установить пакеты или обновить Airflow. Пожалуйста, не забудьте сохранить эти сценарии. Также имейте в виду, что в случаях, когда вы выполняете чистые задачи Python, вы можете использовать функции Python Virtualenv, которые будут динамически создавать и устанавливать зависимости Python во время выполнения. Airflow 2.8.0 также позволяет кэшировать виртуальные визиты.`

##  Особый случай - добавление зависимостей через файл requirements.txt

Обычный случай для пользовательских изображений, когда вы хотите добавить набор требований к нему - обычно хранится в файле requirements.txt. Для разработки вы можете добавлять его динамически, когда начинаете оригинальный образ, но это имеет ряд побочных эффектов (например, ваши контейнеры будут запускаться гораздо медленнее - каждая дополнительная зависимость еще больше замедлит время запуска ваших контейнеров). Также это совершенно не нужно, потому что docker compose имеет встроенный рабочий процесс разработки. Вы можете - после предыдущей главы, автоматически создавать и использовать свой собственный образ. Если вы хотите добавить свой собственный файл требований, вам следует выполнить следующие шаги:

    1. Комментарий к `image: ...` line и удалять комментарий из `build: .` line в `docker-compose.yaml` файле. Соответствующая часть вашего docker-compose файла должна выглядеть примерно так (используйте корректную версию образа):

    `#image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:2.10.5}
     build: .`

    2. Создать докерфайл в той же папке, где находится ваш `docker-compose.yaml` cо схожим наполнением представленым ниже:

    `FROM apache/airflow:2.10.5
     ADD requirements.txt .
     RUN pip install apache-airflow==${AIRFLOW_VERSION} -r requirements.txt`


Рекомендуется установить `apache-airflow` в той же версии, что и исходное изображение. Таким образом вы можете быть уверены, что `pip` не будет пытаться деградировать или обновить `apache airflow` при установке других требований, что может произойти в случае добавления зависимости, которая противоречит версии `apache-airflow`, которую вы используете.

     3. Поместите файл `requirements.txt` в тот же каталог.

Запускайте `docker compose build` для построения изображения, или добавьте флаг `--build `для `docker compose up` или `docker compose run` команды для автоматического построения изображения.

## Особый случай - добавление пользовательского файла конфигурации

Если у вас есть пользовательский файл конфигурации и вы хотите использовать его в вашем экземпляре Airflow, вам нужно выполнить следующие шаги:

    1. Удалить комментарий из `AIRFLOW_CONFIG: '/opt/airflow/config/airflow.cfg'` в `docker-compose.yaml` файле.

    2. Поместите свой файл `airflow.cfg` в локальную папку config.

    3. Если у вашего файла конфигурации другое имя, нежели `airflow.cfg`, измените имя файла в `AIRFLOW_CONFIG: '/opt/airflow/config/airflow.cfg'`

## Сеть

Если вы хотите использовать Airflow локально, ваши DAGs могут попытаться подключиться к серверам, которые работают на узле. Чтобы достичь этого, необходимо добавить дополнительную конфигурацию в `docker-compose.yaml`. Например, на Linux конфигурации в разделе `services: airflow-worker` должна быть добавлено `extra_hosts: - "host.docker.internal:host-gateway"`; и использоваться `host.docker.internal` вместо `localhost`. Эта конфигурация варьируется на разных платформах. Для получения дополнительной информации, ознакомьтесь с документацией по Docker для [Windows](https://docs.docker.com/desktop/features/networking/#use-cases-and-workarounds) и Mac.

## Отладка Airflow внутри контейнера докера с помощью PyCharm

Предварительные условия: Создать проект в PyCharm и скачать ([docker-compose.yaml](https://airflow.apache.org/docs/apache-airflow/2.10.5/docker-compose.yaml)).

Шаги:

    1. Изменить docker-compose.yaml. Добавить следующий блок в раздел, посвященный услугам:
    
    `airflow-python:
          <<: *airflow-common
          profiles:
              - debug
          environment:
              <<: *airflow-common-env
          user: "50000:0"
          entrypoint: [ "/bin/bash", "-c" ]`

ПРИМЕЧАНИЕ: Этот фрагмент кода создает новую службу под названием "airflow-python" специально для интерпретатора PyCharm в Python. На системе Linux, если вы выполнили команду `echo -e "AIRFLOW_UID=$(id -u)" > .env`, Вам нужно установить имя пользователя: `user: "50000:0"` в `airflow-python` service, чтобы избежать ошибки `Unresolved reference 'airflow'` в PyCharm.
  
    2. Настроить интерпретатор PyCharm

        * Откройте PyCharm и перейдите в **Settings** > **Project: <Your Project Name>** > **Python Interpreter**.

        * Нажмите кнопку **Add Interpreter**  и выберите **On Docker Compose**.

        * В поле Файл конфигурации (Configuration file) выберите ваш `docker-compose.yaml` файл.

        * В поле Сервис (Service field) выберите недавно добавленную службу `airflow-python`.

        * Нажмите "Далее" (Next) и следуйте инструкциям, чтобы завершить настройку.

![](https://github.com/sx118828/Running-Airflow-in-Docker/blob/main/1_add_container_python_interpreter.png)

Создание индекса интерпретатора может занять некоторое время. 
    
    З. Добавить `exec` в docker-compose/command и действия в python service

![](https://github.com/sx118828/Running-Airflow-in-Docker/blob/main/2_docker-compose-pycharm.png)

После настройки вы можете отладить код Airflow в контейнерной среде, имитируя локальную установку.

## FAQ: Часто задаваемые вопросы

`ModuleNotFoundError: No module named 'XYZ'`

Файл Docker Compose использует последний образ Airflow ([apache/airflow](https://hub.docker.com/r/apache/airflow)). Если вам нужно установить новую библиотеку или системную библиотеку Python, вы можете её [настроить и расширить](https://airflow.apache.org/docs/docker-stack/index.html).

## Что дальше?

Теперь вы можете перейти к разделу [Учебные пособия](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/index.html) для получения дополнительных примеров или к разделу [Руководства по работе](https://airflow.apache.org/docs/apache-airflow/stable/howto/index.html), если вы готовы замарать руки.

## Переменные среды (Environment), поддерживаемые Docker Compose

Не путайте имена переменных с аргументами сборки, установленными при сборке образа. По умолчанию `AIRFLOW_UID` - `50000`, когда создается изображение. Со своей стороны, следующие переменные среды можно установить при работе контейнера, используя - например - результат команды `id -u`, которая позволяет использовать динамический пользовательский ID, неизвестный на момент создания образа.




