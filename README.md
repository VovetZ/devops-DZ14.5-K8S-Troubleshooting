# devops-DZ14.5-K8S-Troubleshooting

# Домашнее задание к занятию Troubleshooting

### Цель задания

Устранить неисправности при деплое приложения.

### Чеклист готовности к домашнему заданию

1. Кластер K8s.

### Задание. При деплое приложение web-consumer не может подключиться к auth-db. Необходимо это исправить

1. Установить приложение по команде:

```shell
kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
```

2. Выявить проблему и описать.
3. Исправить проблему, описать, что сделано.
4. Продемонстрировать, что проблема решена.

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

-----

# Ответ

# Задание 1

1. Применил манифест

```bash
vk@vkvm:~/14_4$ kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
Error from server (NotFound): error when creating "https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml": namespaces "web" not found
Error from server (NotFound): error when creating "https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml": namespaces "data" not found
Error from server (NotFound): error when creating "https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml": namespaces "data" not found
```

Наблюдаем ошибку, что не найдены пространства имен (`namespace`)

2. Создал пространства имен (`namespace`)

```bash
vk@vkvm:~/14_4$ kubectl create namespace web
namespace/web created
vk@vkvm:~/14_4$ kubectl create namespace data
namespace/data created
```

3. Снова применил манифест

```bash
vk@vkvm:~/14_4$ kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
deployment.apps/web-consumer created
deployment.apps/auth-db created
service/auth-db created
```

**Ошибок нет, развёртывание успешно**

4. Проверил созданные артефакты в пространствах имён `web`, `data`

```bash
vk@vkvm:~/14_4$ kubectl get all -n web -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
pod/web-consumer-577d47b97d-88pl6   1/1     Running   0          36s   10.233.71.13   node3   <none>           <none>
pod/web-consumer-577d47b97d-hzlfc   1/1     Running   0          36s   10.233.74.77   node4   <none>           <none>

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                    SELECTOR
deployment.apps/web-consumer   2/2     2            2           36s   busybox      radial/busyboxplus:curl   app=web-consumer

NAME                                      DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                    SELECTOR
replicaset.apps/web-consumer-577d47b97d   2         2         2       36s   busybox      radial/busyboxplus:curl   app=web-consumer,pod-template-hash=577d47b97d
vk@vkvm:~/14_4$ kubectl get all -n data -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
pod/auth-db-795c96cddc-hkvsw   1/1     Running   0          76s   10.233.75.16   node2   <none>           <none>

NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/auth-db   ClusterIP   10.233.2.201   <none>        80/TCP    76s   app=auth-db

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/auth-db   1/1     1            1           76s   nginx        nginx:1.19.1   app=auth-db

NAME                                 DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
replicaset.apps/auth-db-795c96cddc   1         1         1       77s   nginx        nginx:1.19.1   app=auth-db,pod-template-hash=795c96cddc
```

5. Проверил логи подов в развёртывании `web-consumer`

```bash
vk@vkvm:~/14_4$ kubectl logs deployment/web-consumer -n web
Found 2 pods, using pod/web-consumer-577d47b97d-88pl6
curl: (6) Couldn't resolve host 'auth-db'
curl: (6) Couldn't resolve host 'auth-db'
curl: (6) Couldn't resolve host 'auth-db'
......
```

Увидел, что у подов в развертывании `web-consumer` ошибка разрешения  имени `auth-db`

6. Проверил в `kubectl` разрешение имени `auth-db`

```bash
vk@vkvm:~/14_4$ kubectl -n web exec web-consumer-577d47b97d-88pl6 -- curl auth-db
curl: (6) Couldn't resolve host 'auth-db'
command terminated with exit code 6
```

Далее проверил разрешение по полному имени (FQDN) `auth-db.data.svc.cluster.local`

```bash
vk@vkvm:~/14_4$ kubectl -n web exec web-consumer-577d47b97d-88pl6 -- curl  -I auth-db.data.svc.cluster.local
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0   612    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
HTTP/1.1 200 OK
Server: nginx/1.19.1
Date: Sat, 15 Jul 2023 13:29:24 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 07 Jul 2020 15:52:25 GMT
Connection: keep-alive
ETag: "5f049a39-264"
Accept-Ranges: bytes
```

Убедился, что разрешение по полному имени работает.
Полное имя включает в себя пространство имён `data` для пода `auth-db`.

7. Отредактировал манифест для пода развёртывания `web-consumer`, добавил полное имя до пода `auth-db`. После сохранения поды должны пересоздаться.

```bash
vk@vkvm:~/14_4$ kubectl edit -n web deployments/web-consumer
deployment.apps/web-consumer edited
```

8. Проверил логи

```bash
kubectl logs deployment/web-consumer -n web
Found 2 pods, using pod/web-consumer-7f687d84fc-mgwst
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0   114k      0 --:--:-- --:--:-- --:--:--  298k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
```bash
vk@vkvm:~/14_4$ kubectl logs deployment/auth-db -n data
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
10.233.71.13 - - [15/Jul/2023:13:29:08 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.35.0" "-"
10.233.71.13 - - [15/Jul/2023:13:29:24 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.35.0" "-"
10.233.71.13 - - [15/Jul/2023:13:29:47 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.35.0" "-"
10.233.74.78 - - [15/Jul/2023:13:31:13 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.35.0" "-"
10.233.75.17 - - [15/Jul/2023:13:31:17 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.35.0" "-"
10.233.74.78 - - [15/Jul/2023:13:31:18 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.35.0" "-"
```

Убедился, что доступ из `web-consumer` в `auth-db` работает. 

**Проблема устранена**
