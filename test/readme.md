# Развертывание NGINX

- Для изменения **namespace** измените в файлах metadata: namespace: Ваш_namespace
- Поставить Nginx POD:
```shell
kubectl apply -f simple-pod.yaml
```
- Подключить сервис:
```shell
kubectl apply -f nginx-service.yaml
```
- В файле `nginx-ingress.yaml` укажите ваш домен: `rules:- host: Ваш_Домен`
- Подключить Ingress:
```shell
kubectl apply -f nginx-ingress.yaml
```


