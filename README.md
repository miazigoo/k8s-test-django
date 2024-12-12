# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema)


## Запуск в кластере Kubernetes (minikube)
 - Устанавливаем и запускаем [minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/) 
 - Создаем docker image приложения
 ```shell
eval $(minikube docker-env)
docker build -t django_app:latest ./backend_main_django
```
 - Запускаем в кластере БД PostgreSQL с помощью  [helm](https://helm.sh/ru/docs/intro/install/)
```shell
# Запуск PostgreSQL
helm install pg-db --set auth.postgresPassword=<db_password> oci://registry-1.docker.io/bitnamicharts/postgresql
!!Важно - после запуска этой команды в терминале будет выведена подробная информация с командами ниже . Команды приведенные ниже указанны для образца .
# Делаем экспорт пароля от админ-пользователя БД в переменную окружения:
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default pg-db-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

# Создаем `pod` с утилитой `psql` и выполнить в ней команду подключения к БД (после запуска # команды дождитесь загрузки и появления `postgres=#` ):
kubectl run pg-db-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:15.3.0-debian-11-r7 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host pg-db-postgresql -U postgres -d postgres -p 5432

# Создаем пользователя БД (и пароль) для подключения приложения:
CREATE ROLE <username> WITH LOGIN ENCRYPTED PASSWORD '<put-your-password-here>';

# Создаем базу для работы приложения, владельцем которой будет выбранный пользователь:
CREATE DATABASE <database-name> OWNER <username>;
```
Устанавливаем Ingress, например Contour:
```shell
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

- В директории kubernetes создаем файл конфигурации `config.yaml` с настройками для приложения:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-app-config
  labels:
    env: dev
    author: <you_name>
data:
  SECRET_KEY: "django_secret_key"
  DEBUG: "False"
  ALLOWED_HOSTS: "127.0.0.1,star-burger.test"
  DATABASE_URL: "postgres://<db_user>:<db_password>@pg-db-postgresql:5432/<db_name>"

```
 - Применяем файл настроек:
```shell
kubectl apply -f k8s/config.yaml
```
 - Стартуем остальные процессы:
 ```shell
 kubectl apply -f k8s/deploy.yaml, k8s/service.yaml, k8s/ingress.yaml, k8s/clearsessions.yaml
```
 - Если надо запустить миграции
 ```shell
 kubectl apply -f k8s/migrate.yaml
```
 - Создаем суперпользователя приложения
 ```shell
 kubectl exec -it <django-deploy-pod-name> -- python manage.py createsuperuser
```
 - Для того чтобы к сайту можно было получить доступ по DNS имени выполняем
 ```shell
 echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```

Проверить работу всех приложений:
```shell
kubectl get deployments.apps,pods,service,configmaps,ingress,cronjobs.batch
```
Для изменения настроек приложения (после внесения изменений в `ConfigMap`)
```shell
kubectl apply -f k8s/config.yaml && kubectl rollout restart deployment django-deploy
```
