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
