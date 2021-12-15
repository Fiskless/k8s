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

Для тонкой настройки используйте переменные окружения. Список доступных переменных можно найти внутри файла `docker-compose.yml`.

## Как запустить prod-версию на Minikube:

Установите [VirtualBox](https://www.virtualbox.org/), [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/),
поднимите кластер [minikube](https://minikube.sigs.k8s.io/docs/drivers/virtualbox/)

Соберите образ django в кластере:
```shell-session
$ minikube image build -t django_app .
```

Создайте фал ConfigMap.yaml такого вида:
```shell-session
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: django-app-config
      labels:
        app: django-app
    data:
      DEBUG: "False"
      DATABASE_URL: your_db_data
```

Загрузите ConfigMap в кластер:
```shell-session
$ kubectl apply -f ConfigMap.yaml
```

Разверните сайт:
```shell-session
$ kubectl apply -f django-deployment-nodeport.yaml
```

Для настройки [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) сделайте следующее:

Установите Ingress:
```shell-session
$ kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

Задайте адрес для домена в /etc/hosts/ на вашей машине в виде:

```shell-session
virtual_machine_ip  star-burger.test
```

Создайте сервис и настройте Ingress:
```shell-session
$ kubectl apply -f ingress-hosts.yaml
$ kubectl apply -f django-deployment-clusterip.yaml
```

Перейдите на `http://star-burger.test`, там появится ваш сайт

Добавьте ежемесячное удаление [сессий](https://docs.djangoproject.com/en/3.1/topics/http/sessions/#clearing-the-session-store) Django:
```shell-session
$ kubectl apply -f clearsessions-cronjob.yaml
```


## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

