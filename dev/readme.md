# Запуск в кластере Kubernetes 
### Необходимо иметь готовый кластер и БД, например в [cloud.reg.ru](https://cloud.reg.ru)

- Установите [Helm](https://helm.sh/docs/intro/install/).
- Устанавливаем Ingress-контроллер, например Nginx, с помощью команды:
```shell
helm upgrade --install ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx --create-namespace
```
- Значения необходимо перевести значения в закодированный вид (Base64)
```yaml
SECRET_KEY: "django_secret_key"
DATABASE_URL: "postgres://<db_user>:<db_password>@pg-db-postgresql:5432/<db_name>"
```
- В директории kubernetes создаем файл конфигурации `secret.yaml` с настройками для приложения:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secret-config
  namespace: edu-practical-goldwasser
  labels:
    author: <you_name>
type: Opaque
data:
  SECRET_KEY: UkVQTEFDRV9NRQ==
  DATABASE_URL: cG9zdGdyZXM6Ly9iMDBiczpRd3NBengkMjAwMEA3OS4xNzQuODguMTg6MTUyNzcvYjAwYnM=
```
- Или вместо создания файла `secret.yaml` можно ввести командуЖ
```shell
kubectl create secret generic django-secret-config --from-literal=SECRET_KEY=$secret_key --from-literal=DATABASE_URL=postgres://<db_user>:<db_password>@pg-db-postgresql:5432/<db_name> 


```
 - Применяем файл настроек:
```shell
kubectl apply -f dev/config.yaml
```
- Применяем файл секретных настроек, если был создан:
```shell
kubectl apply -f dev/secret.yaml
```
 - Стартуем остальные процессы:
 ```shell
 kubectl apply -f dev/deploy.yaml, dev/service.yaml, dev/ingress.yaml, dev/clearsessions.yaml
```
 - Если надо запустить миграции
 ```shell
 kubectl apply -f dev/migrate.yaml
```
 - Создаем суперпользователя приложения
 ```shell
 kubectl exec -it <django-deploy-pod-name> -- python manage.py createsuperuser
```

Проверить работу всех приложений:
```shell
kubectl get deployments.apps,pods,service,configmaps,ingress,cronjobs.batch
```
Для изменения настроек приложения (после внесения изменений в `ConfigMap` или `Secret`)
```shell
kubectl apply -f dev/config.yaml && kubectl rollout restart deployment django-deploy
```