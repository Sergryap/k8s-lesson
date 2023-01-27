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


## Как запустить сайт в кластере minikube

У вас должны быть установлены `minikube` и `kubectl`

#### Запустите minikube:
```sh
minikube start
```
#### Включите надстройку входа minikube с помощью следующей команды:
```sh
minikube addons enable ingress
```

#### Создайте файл для переменных окружения
Например `.env`, содержащий все необходимые переменные для работы приложения:
```
SECRET_KEY=<секретный ключ вашего проекта>
DEBUG=False
DATABASE_URL=postgres://test_k8s:OwOtBep9Frut@<host>:5432/test_k8s
ALLOWED_HOSTS=127.0.0.1,localhost,<ip вашего кластера>
```
#### Создайте ConfigMap для вашего проекта:
```sh
kubectl create configmap django-app-config --from-env-file=backend_main_django/.env
```
Если в дальнейшем потребуется обновить ConfigMap, выполните сначала удаление предыдущей версии:
```sh
kubectl delete configmap django-app-config
```
#### Создайте Deployment и Service для вашего приложения django:
```sh
kubectl apply -f deployment-django-app.yaml
```
#### Добавьте службу TCP в контроллер входящего трафика nginx, выполните следующую команду:
```sh
kubectl patch configmap tcp-services -n ingress-nginx --patch '{"data":{"80":"default/django-app-service:80"}}'
```
Где:
- `80`: порт, который ваша служба должна прослушивать из-за пределов виртуальной машины minikube
- `default`: пространство имен, в котором установлена ваша служба
- `django-app-service`: название сервиса 

Мы можем убедиться, что наш ресурс был исправлен с помощью следующей команды:
```sh
kubectl get configmap tcp-services -n ingress-nginx -o yaml
```
#### Настройте правила для воходящего трафика для вашего ingress:
```sh
kubectl apply -f ingress-django-app.yaml
```
#### Вы можете проветить состояние добавленного вами Ingress:
```sh
kubectl get ingress
```
![Снимок экрана от 2023-01-24 22-42-48](https://user-images.githubusercontent.com/99894266/214368116-a365f65d-e7b4-4c90-8f79-0d241dcb8148.png)
#### Проверьте, что сервисы спрятаны за ClusterIP:
```sh
kubectl get service
```
![Снимок экрана от 2023-01-24 22-56-04](https://user-images.githubusercontent.com/99894266/214370900-bda0512c-c99f-4272-a76b-a0f6d8e458bf.png)

#### Обновите файл /etc/hosts для маршрутизации запросов от star-burger.test к экземпляру minikube:
```sh
echo "$(minikube ip) start-burger.test" | sudo tee -a /etc/hosts
```
#### Сайт будет доступен по ссылке: http://star-burger.test с вашей локальной машины:
![Снимок экрана от 2023-01-24 22-49-25](https://user-images.githubusercontent.com/99894266/214369620-4682c2f5-c3da-4b18-b7da-fa454e6daf43.png)
#### Настройте регулярное удаление сессий
Для этого выполните команду по созданию регулярно повторяющейся задачи CronJob:
```sh
kubectl create -f clearsessions-cronjob.yaml
```
