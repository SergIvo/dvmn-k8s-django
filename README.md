# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Запуск в кластере Kubernetes

Чтобы запустить сайт в кластере, необходимо указать переменные окружения в манифесте `django-config.yaml`:
```yaml
data:
  SECRET_KEY: <секретный ключ Django>
  DEBUG: <"False" или "True"> 
  ALLOWED_HOSTS: star-burger.test
  DATABASE_URL: <адрес для подключения к базе данных PostgreSQL. В качестве хотса в ней должен быть указан адрес сервиса кластера для внешних подключений, например host.minikube.internal в Minikube>
```

Когда все переменные указаны, нужно добавить `configMap` в кластер. Это можно сделать следующей командой:
```
kubectl apply -f kubernetes/django-config.yaml
```
Для того, чтобы сайт можно было открыть в браузере, необходимо запустить и настроить Ingress. Запустить Ingress в Minikube можно следующей командой:
```
minikube addons enable ingress
```
После этого необходимо применить настройки Ingress:
```
kubectl apply -f kubernetes/ingress-hosts.yaml
```
После применения настроек сайт будет доступен в браузере по адресу `star-burger.test`. Запустить сайт можно следующией командой:
```
kubectl apply -f kubernetes/django-deploy.yaml
```
Чтобы применить миграции Django к базе данных, воспользуйтесь следующей командой:
```
kubectl apply -f kubernetes/django-migrate.yaml
```
Также, для автоматического ежемесячного удаления сессий Django создайте запланированную задачу следующей командой:
```
kubectl apply -f kubernetes/django-clearsessions.yaml
```
