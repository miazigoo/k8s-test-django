# Описание команд для **k8s**

- Для сокращения названий команд используйте `alias`, Например:
```shell
alias kub=kubectl
```
- Запуск _**YAML**_ файла:
```shell
kubectl apply -f - k8s/test.yaml
```
- Показать все поды:
```shell
kubectl get pods
```
- Подробная информация о поде:
```shell
kubectl describe pod test-pod
```
- По аналогии с `docker exec` мы можем запускать команды в контейнерах через `kubectl exec`, в том числе в интерактивном режиме:
```
$ kubectl exec test-pod -- whoami
root
$ kubectl exec -it test-pod -- /bin/bash
root@test-pod:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@test-pod:/# 
```

- Получить локальный доступ командой `kubectl port-forward`. Например:
```
$ kubectl port-forward pod/test-pod 7080:8080
Forwarding from 127.0.0.1:7080 -> 8080
Forwarding from [::1]:7080 -> 8080
```

Теперь мы можем открыть [http://localhost:7080/](http://localhost:7080/) и увидеть стандартный ответ `http.server`

- Мы можем посмотреть логи контейнера командой `kubectl logs`. Например:

```
$ kubectl logs test-pod
127.0.0.1 - - [03/Jul/2023 14:24:19] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [03/Jul/2023 14:24:19] code 404, message File not found
127.0.0.1 - - [03/Jul/2023 14:24:19] "GET /favicon.ico HTTP/1.1" 404 -
```


- Для создания стабильного, персистентного набора подов `Kubernetes` 
предоставляет другой низкоуровневый тип ресурса — `ReplicaSet`. 
Репликасет позволяет указать **а)** количество желаемых подов и **б)** шаблон, по которому клепать эти поды. 
Так будет выглядеть описание репликасета, поднимающего два пода с бэкендом:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: backend-replicaset
spec:
  replicas: 2  # сколько подов нужно держать поднятыми

  # template — шаблон для создания подов. Содержимое по синтаксису
  # аналогично описанию пода (см. пример выше), с небольшими различиями:
  # • apiVersion и kind не нужно указывать — и так понятно, что
  #   это под в той же версии API, что и репликасет;
  # • также не нужно указывать name — он будет генерироваться
  #   движком Kubernetes исходя из названия репликасета.
  template:
    metadata:

      # labels — это произвольные key-value пары, хранящие 
      # метаинформацию о поде. Выбор ключа app и значения 
      # backend-app произволен. Но этот лейбл потребуется нам ниже!
      labels:
        app: backend-app

    spec:
      containers:
        - name: backend-container

          # Способ указать minikube использовать ранее собранные
          # нами локально образы. Без настройки imagePullPolicy
          # Kubernetes будет пытаться тянуть образ из интернета 
          # (и, конечно, не сможет его найти и сфейлится).
          image: docker.io/habr-app-demo/backend:latest
          imagePullPolicy: Never

          ports:
            - containerPort: 40001

  # selector — это способ для репликасета понять, какие поды
  # из числа уже существующих в кластере относятся к нему. Поскольку
  # мы прописали в шаблоне выше лейбл app: backend-app, мы точно
  # знаем, что все поды с таким лейблом порождены этим репликасетом.
  # Репликасет будет пользоваться этим селектором, чтобы понять,
  # сколько он уже насоздавал подов и сколько ещё нужно, чтобы 
  # добиться количества реплик, указанного выше в поле replicas.
  selector:
    matchLabels:
      app: backend-app
```


- У репликасета есть минус — неудобно обновлять поды: нам бы хотелось, чтобы если 
мы обновим версию бэкенда, она выкатывалась постепенно, под за подом; с репликасетами 
же нам придётся вручную создавать новый репликасет с актуальной версией и либо разом 
удалять старый (что под нагрузкой довольно опасно), либо долгой многоходовочкой менять `replicas` 
в обоих репликасетах, сдувая старый и надувая новый. К счастью, есть более высокоуровневый компонент, 
умеющий заниматься ровно этим.


- Для удобного развёртывания сервисов в `Kubernetes` есть ресурс `Deployment`, 
для создания которого нужно указать все те же параметры, что и для репликасета, 
плюс (опционально) настройки той самой плавной выкатки — например, какое максимальное 
количество подов может быть в переходном состоянии в каждый момент времени.


- Описание простейшего деплоймента с бэкендом практически не отличается от репликасета:
```yaml
# k8s/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: backend-app
    spec:
      containers:
        - name: backend-container
          image: docker.io/habr-app-demo/backend:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 40001
  selector:
    matchLabels:
      app: backend-app
```
- Давайте посмотрим на простейший StatefulSet, поднимающий одну реплику MongoDB:

```yaml
# k8s/mongodb-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-statefulset
spec:
  serviceName: mongodb-service
  replicas: 1  # сколько подов требуется в стейтфулсете
  template:
    metadata:
      labels:
        # label нужен из тех же соображений, что и в деплойменте.
        app: mongodb-app
    spec:
      containers:
        - name: mongodb-container
          image: mongo:6.0.6

          # Выставляем наружу дефолтный порт монги.
          ports:
            - containerPort: 27017
              name: mongodb-cli
          volumeMounts:
            - mountPath: "/data/db"
              name: mongodb-pvc

  # selector нужен из тех же соображений, что и в деплойменте.
  selector:
    matchLabels:
      app: mongodb-app
  volumeClaimTemplates:
    - metadata:
        name: mongodb-pvc
      spec:
        accessModes:
          - ReadWriteOnce # можно читать/писать только на одной ноде
        resources:
          requests:
            storage: 100Mi
```
- Для сетевого доступа к подам нам потребуется ещё одна абстракция — Service. Описание сервиса состоит из селектора подов, к которым нужно обеспечить доступ, и типа доступа, которых есть несколько. Поскольку нам нужен внутренний доступ к одному конкретному поду в StatefulSet, нас устроит самый простой тип сервиса, так называемый headless service — без балансировки нагрузки и выделения статического IP для доступа извне:
```yaml
# k8s/mongodb-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  # ClusterIP — самый простой тип сервиса, который 
  # позволяет подам связываться друг с другом в рамках
  # кластера, но абсолютно никак не виден снаружи:
  type: ClusterIP
  clusterIP: None
  selector:
    app: mongodb-app
```




