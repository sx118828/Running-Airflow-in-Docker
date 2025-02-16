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

`mkdir -p ./dags ./logs ./plugins ./config
echo -e "AIRFLOW_UID=$(id -u)" > .env`

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

`CONTAINER ID   IMAGE                   COMMAND                  CREATED          STATUS                    PORTS                              NAMES`

`247ebe6cf87a   apache/airflow:2.10.5   "/usr/bin/dumb-init …"   3 minutes ago    Up 3 minutes (healthy)    8080/tcp                           compose_airflow-worker_1`

`ed9b09fc84b1   apache/airflow:2.10.5   "/usr/bin/dumb-init …"   3 minutes ago    Up 3 minutes (healthy)    8080/tcp                           compose_airflow-scheduler_1`

`7cb1fb603a98   apache/airflow:2.10.5   "/usr/bin/dumb-init …"   3 minutes ago    Up 3 minutes (healthy)    0.0.0.0:8080->8080/tcp             compose_airflow-webserver_1`

`74f3bbe506eb   postgres:13             "docker-entrypoint.s…"   18 minutes ago   Up 17 minutes (healthy)   5432/tcp                           compose_postgres_1`

`0bd6576d23cb   redis:latest            "docker-entrypoint.s…"   10 hours ago     Up 17 minutes (healthy)   0.0.0.0:6379->6379/tcp             compose_redis_1`

`* $ docker ps`
`* CONTAINER ID   IMAGE                   COMMAND                  CREATED          STATUS                    PORTS                              NAMES`
`* 247ebe6cf87a   apache/airflow:2.10.5   "/usr/bin/dumb-init …"   3 minutes ago    Up 3 minutes (healthy)    8080/tcp                           compose_airflow-worker_1`
`* ed9b09fc84b1   apache/airflow:2.10.5   "/usr/bin/dumb-init …"   3 minutes ago    Up 3 minutes (healthy)    8080/tcp                           compose_airflow-scheduler_1`
`* 7cb1fb603a98   apache/airflow:2.10.5   "/usr/bin/dumb-init …"   3 minutes ago    Up 3 minutes (healthy)    0.0.0.0:8080->8080/tcp             compose_airflow-webserver_1`
`* 74f3bbe506eb   postgres:13             "docker-entrypoint.s…"   18 minutes ago   Up 17 minutes (healthy)   5432/tcp                           compose_postgres_1`
`* 0bd6576d23cb   redis:latest            "docker-entrypoint.s…"   10 hours ago     Up 17 minutes (healthy)   0.0.0.0:6379->6379/tcp             compose_redis_1`

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



