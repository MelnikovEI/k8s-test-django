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



# Деплой в Minikube (Windows, VirtualBox)

1. Установите  
#### [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)  
Проверить результат:
```sh
kubectl version --client
```
#### [minikube](https://minikube.sigs.k8s.io/docs/start/)  
Проверить результат:
```sh
minikube version
```
#### [VirtualBox](https://www.virtualbox.org/)
#### [Docker](https://docs.docker.com/get-docker/)  
Если Вы на windows, запустите VirtualBox, Docker desktop (от имени администратора).  

"Поднимите" локальный K8s Cluster
```sh
minikube start --driver=virtualbox --no-vtx-check
```
Проверить результат:
```sh
kubectl cluster-info
```
`-> Kubernetes control plane is running at https://192.168.59.101:8443
CoreDNS is running at https://192.168.59.101:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.`

2. Соберите образ с приложением `django`  
Перейдите в папку проекта^
```sh
cd 'ваш путь до папки проекта'\k8s-test-django\
```
```sh
minikube image build ./backend_main_django/ -t django_app:0.0.1
```
Если возникла ошибка "Unable to resolve the current Docker CLI context "default": context "default": context not found", выполните команду:
```sh
docker context use default
```
Проверить результат:
```sh
minikube image ls
```
`-> ... docker.io/library/django_app:0.0.1 ...`

3. Установите [Helm](https://helm.sh/)

4. Разверните PostgreSQL в кластере
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```
```sh
helm install postgres bitnami/postgresql
```
Получите пароль:  
```sh
kubectl get secret --namespace default postgres-postgresql -o jsonpath="{.data.postgres-password}"
```
Результат преобразуйте в текст:
```sh
powershell "[Text.Encoding]::UTF8.GetString([convert]::FromBase64String('ПАРОЛЬ_ИЗ_ПРЕДЫДУЩЕГО_ШАГА'))"
```
Войдите в postgres:
```sh
kubectl run postgres-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:16.0.0-debian-11-r13 --env="PGPASSWORD=ПАРОЛЬ_ИЗ_ПРЕДЫДУЩЕГО_ШАГА" --command -- psql --host postgres-postgresql -U postgres -d postgres -p 5432
```
Создайте БД и пользователя в PostgreSQL:
```sh
postgres=#
postgres=# create database test_k8s;
postgres=# create user test_k8s with encrypted password 'OwOtBep9Frut';
postgres=# grant all privileges on database test_k8s to test_k8s;
postgres=# ALTER DATABASE test_k8s OWNER TO test_k8s;
```

5. Получите IP адрес из команды `kubectl cluster-info` и запишите его в конец файла `c:\windows\system32\drivers\etc\hosts`:  
например: `192.168.59.103 star-burger.test`

6. Создайте dj-config  
Создайте configmap в кластере:  
```sh
kubectl create configmap dj-config --from-file=.\kubernetes\dj-config.yaml
```
Проверить и при необходимости внести изменения:
```sh
kubectl edit configmap dj-config
```
```sh
kubectl apply -f .\kubernetes\django-app-deploy-service.yaml
```
```sh
kubectl rollout restart deployment dj-deployment
```
Проверить:  
```sh
kubectl get pods
```  
```sh
kubectl get service
```

7. Активируйте `ingress`
```sh
minikube addons enable ingress
kubectl apply -f .\kubernetes\ingress-hosts.yaml
```
Проверить:
```sh
kubectl get ingress
```

8. Примените миграции
```sh
kubectl apply -f .\kubernetes\migrate.yaml
```
Проверить:
```sh
kubectl get pods
```
```sh
kubectl logs django-migrate-fqhgj
```
Удалите выполненный job:
```sh
kubectl delete job django-migrate
```

9. Активируйте автоматический `clearsessions`
```sh
kubectl apply -f .\kubernetes\django-clearsessions.yaml
```
Проверить:
```kubectl get cronjob```

10. Открыть `http://star-burger.test/` в браузере. Должна появиться страница `Администрирование Django`
