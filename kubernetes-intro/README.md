# neonlight911_platform
neonlight911 Platform repository

## Знакомство с Kubernetes, основные понятия и архитектура

* Установлен и запущен kubernetes cluster (cloud)
* Настроено рабочее окружение

### Задание: Разберитесь почему все pod в namespace kube-system восстановились после удаления
Запуск агента kubelet реализован как дочерний сервис **systemd**:

~~~
$ systemctl status kubelet.service
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Thu 2021-05-06 19:13:40 UTC; 14h ago
       Docs: https://kubernetes.io/docs/home/
   Main PID: 832086 (kubelet)
      Tasks: 16 (limit: 2286)
     Memory: 50.9M
     CGroup: /system.slice/kubelet.service
             └─832086 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.4.1
~~~

На основании конфигурационных файлов:

~~~
$ ls /etc/kubernetes/*.conf 
admin.conf  controller-manager.conf kubelet.conf  scheduler.conf
~~~

Где и описаны системные **pods**
Если остановить данный сервис, автовосстановление перестанет работать

~~~
systemctl stop kubelet.service
docker rm -f $(docker ps -a -q)
~~~

Для возобновления работы автовосстановления, достаточно снова запустить сервис агента **kubelet**

~~~~
systemctl start kubelet.service
~~~~

**core-dns** запущен посредством конфигурации развёртывания (Deployment)

~~~
$ kubectl get deployments -n kube-system
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           45h
~~~

Параметр **replicas** регламентирует кол-во реплик, которые кластер будет поддерживать в запущеном состоянии

~~~
$ kubectl get deployments -n kube-system -o yaml|grep replicas
    replicas: 2
    replicas: 2
~~~

**kube-apiserver** аналолгично **etcd**, **kube-controller-manager**, **kube-scheduler** запускается **kubelet**'ом из **/etc/kubernetes/manifests/**

~~~
$ ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
~~~

**kube-proxy** управляется и создается посредством **Daemonset** (будет запускать не более одной реплики на каждом узле)

~~~
$ kubectl -n kube-system get pod -l k8s-app=kube-proxy -o wide
NAME               READY   STATUS    RESTARTS   AGE     IP              NODE          NOMINATED NODE   READINESS GATES
kube-proxy-6gnh6   1/1     Running   0          3d18h   XXX.XXX.XX.XX   k8s-master1   <none>           <none>
kube-proxy-lmpnt   1/1     Running   0          3d18h   XXX.XXX.XX.XX   k8s-node1     <none>           <none>
kube-proxy-mzxz8   1/1     Running   0          3d18h   XXX.XXX.XX.XX   k8s-node2     <none>           <none>
~~~

Принадлежномть (**ownerReferences**) контроллеру **DaemonSet**

~~~
$ kubectl -n kube-system get pod kube-proxy-6gnh6 -o yaml|grep "Deployments\|DaemonSet" -B4
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: DaemonSet
~~~

### Задание: Создать Dockerfile в котором будет описан образ:
* Запускающий web-сервер на порту 8000
* Отдающий содержимое директории /app
* Файл /app/homework.html
* Работающий с UID 1001
* разместить в kubernetes-intro/web 
* Собрать образ и разместить его в DockerHub

https://github.com/otus-kuber-2021-03/neonlight911_platform/blob/kubernetes-intro/kubernetes-intro/web/Dockerfile

Проброс подключения к поду
~~~
kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
~~~
Проверка работоспобности
~~~
$ curl -v http://127.0.0.1:8000/homework.html
*   Trying 127.0.0.1:8000...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8000 (#0)
> GET /homework.html HTTP/1.1
> Host: 127.0.0.1:8000
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.19.10
< Date: Mon, 10 May 2021 22:30:10 GMT
< Content-Type: text/html
< Content-Length: 104
< Last-Modified: Mon, 10 May 2021 21:17:50 GMT
< Connection: keep-alive
< ETag: "6099a2fe-68"
< Accept-Ranges: bytes
< 
<html>
 <head>
 </head>
 <body>
   <h1>Hello World<h1>
   <h1>There is my home work<h1>
 </body>
* Connection #0 to host 127.0.0.1 left intact
</html>
~~~

### Hipster shop
Выясните причину, по которой pod frontend находится в статусе Error

Экспорт манифеста
~~~
kubectl run frontend --image neonligh911/hipster-shop --restart=Never --dry-run -o yaml > frontend-pod.yaml
~~~
Вывод состояния pod'a
~~~
$ kubectl get pod frontend
NAME       READY   STATUS   RESTARTS   AGE
frontend   0/1     Error    0          2m29s
~~~
Вывод журналов pod'a
~~~
$ kubectl logs -f frontend
panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set

goroutine 1 [running]:
main.mustMapEnv(0xc00032e000, 0xb03c4a, 0x1c)
	/go/src/github.com/GoogleCloudPlatform/microservices-demo/src/frontend/main.go:248 +0x10e
main.main()
	/go/src/github.com/GoogleCloudPlatform/microservices-demo/src/frontend/main.go:106 +0x3e9
~~~
В итоге pod не запускался из-за отсутсвия инициализированых переменных окружения

https://github.com/otus-kuber-2021-03/neonlight911_platform/blob/kubernetes-intro/kubernetes-intro/frontend-pod-healthy.yaml
