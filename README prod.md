# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить prod-версию

### 1. Актуализируйте переменные окружения в файле `dj-config.yaml`

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

### 2. Создайте образ приложения.
Соберите образ с приложением `django`    
Перейдите в папку проекта  
```sh
cd 'ваш путь до папки проекта'\k8s-test-django\
```
```sh
docker image build ./backend_main_django/ -t django_app:0.0.1
```
Если возникла ошибка "Unable to resolve the current Docker CLI context "default": context "default": context not found", выполните команду:
```sh
docker context use default
```
Проверить результат:
```sh
docker image ls
```
`-> ... docker.io/library/django_app:0.0.1 ...`

Выложите образ в [dockerhub](https://hub.docker.com/)
```sh
docker push Ваш_репозиторий/django_app:0.0.1
``` 
Запишите ссылку на образ в файле django-app-deploy-service.yaml    
Например:  
```image: Ваш_репозиторий/django_app:0.0.1```

### 3. Создайте configmap в кластере:  
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

### 4. Обновите данные в
- `django-app-deploy-service.yaml`
- `migrate.yaml`
- `django-clearsessions.yaml`

### 5. Примените миграции
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

### 6. Активируйте автоматический `clearsessions`
```sh
kubectl apply -f .\kubernetes\django-clearsessions.yaml
```
Проверить:
```kubectl get cronjob```

### 7. Как выкатить на prod свежую версию сайта?
Создать и залить новую версию образа, см. п.2 `django_app:x.x.x`  
Применить п.4, 5.
