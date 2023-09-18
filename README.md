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

## Как развернуть сайт в Minukube

`minikube`, `kubectl`, `VirtualBox` должны быть установлены

Создайте кластер в minikube:
```sh
minikube start
```

Включите надстройку входа minikube:
```sh
minikube addons enable ingress
```

Запустите PostgreSQL в созданном кластере:
```sh
helm repo add my-repo https://charts.bitnami.com/bitnami
helm install my-release my-repo/postgresql
```

Создайте Secret:
```sh
kubectl create -f secret-app.yaml
```

Создайте Deployment и Service:
```sh
kubectl apply -f deployment-django-app.yaml
```

Настройте правила выходящего трафика для ingress:
```sh
kubectl apply -f ingress-django-app.yaml
```

Запустите команды для настройки базы данных:
```sh
kubectl apply -f django-manage-migrate.yaml<br>
kubectl apply -f django-manage-createsuperuser.yaml
```

Настройте регулярное удаление сессий:
```sh
kubectl create -f clearsessions.yaml
```

Обновите файл /etc/hosts для маршрутизации запросов от star-burger.test:
```sh
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```

Сайт будет доступен по ссылке: http://star-burger.test с вашей локальной машины:

## Как развернуть сайт в Kubernetes
Пример работающего сайта можно посмотреть по [ссылке](https://star-burger.ru/).

Разверните кластер Kubernetes в облаке, например в [VK CLOUD](https://mcs.mail.ru/)

После создания кластера Вам будет предложено сохранить файл конфигурации кластера с названием типа `kubernetes-cluster-1791_kubeconfig.yaml`.

Поместите его у себя на локальной машине в директорию `.../.kube/`

Установите kubectl для взаимодействия с кластером

Установите `helm` для запуска PostgreSQL в кластере: 
```sh
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm install my-db oci://registry-1.docker.io/bitnamicharts/postgresql
```
Создайте файл `secrets.yml` со следующим содержанием:

```
apiVersion: v1
kind: Secret
metadata:
  name: ...
type: Opaque
stringData:
  secret_key: "..."
  debug: "False"
  database_url: "..."
  allowed_hosts: "..."
  superuser_password: "..."
  superuser_email: "..."
```

Запустите комманду для создания секрета в кластере:
```sh
kubectl apply -f secrets.yml
```
Запустите команду для загрузки проекта в кластер:
```sh
kubectl apply -f django-deployment.yml.
```
Запустите команды для настройки базы данных:
```sh
kubectl apply -f django-manage-migrate.yaml<br>
kubectl apply -f django-manage-createsuperuser.yaml
```
