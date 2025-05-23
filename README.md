# Репозиторий для выполнения домашних заданий курса "Инфраструктурная платформа на основе Kubernetes-2025-02" 

1. **HomeWork 1** 

Необходимо создать манифест namespace.yaml для namespace с именем homework.  
Необходимо создать манифест pod.yaml. Он должен описывать под, который: 
* Будет создаваться в namespace homework ○ Будет иметь контейнер, поднимающий веб-сервер на 8000 порту и отдающий содержимое папки /homework внутри этого контейнера.  
* Будет иметь init-контейнер, скачивающий или генерирующий файл index.html и сохраняющий его в директорию /init ○ Будет иметь общий том (volume) для основного и init- контейнера, 
монтируемый в директорию /homework первого и /init второго ○ Будет удалять файл index.html из директории /homework основного контейнера, перед его завершением. 


<details>
  <summary>Ответ</summary>

Основные DNS записи: 
namespace.yaml - создаёт namespace. 
configmap.yaml - заменяет дефолтный конфиг ngix. 
service.yaml - делаем сервис, для проверки работы пода снаружи через NodePort. 
pod.yaml - описываем сам под. 

### запуск
```
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f pod.yaml
 kubectl apply -f service.yaml
```


</details>

2. **HomeWork 2**

Необходимо создать манифест namespace.yaml для namespace с именем homework (уже создан в первом дз) 
Необходимо создать манифест deployment.yaml. Он должен описывать deployment, который будет создаваться в namespace homework 
Запускает 3 экземпляра пода, полностью аналогичных по спецификации прошлому ДЗ. 
В дополнение к этому будет иметь readiness пробу, проверяющую наличие файла /homework/index.html.

Будет иметь стратегию обновления RollingUpdate, настроенную так, что в процессе обновления может быть недоступен максимум 1 под. 
Добавить к манифесту deployment-а спецификацию, обеспечивающую запуск подов деплоймента, только на нодах кластера, имеющих метку homework=true. 

<details>
  <summary>Ответ</summary>

Создаём манифест 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homework-deployment
  namespace: homework
  labels:
    app: homework
spec:
  replicas: 3  # Запускаем 3 экземпляра пода из прошлого ДЗ
  selector:
    matchLabels:
      app: homework
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # В процессе обновления может быть недоступен 1 под
      maxSurge: 1        # Указываем что в процессе обновления можно создавать 1 дполнительный под
  template:
    metadata:
      labels:
        app: homework
    spec:
      nodeSelector:
        homework: "true"  # Указываем, что поды могут запускаться только на нодах с меткой homework
      volumes:
        - name: shared-volume
          emptyDir: {}
        - name: config-volume
          configMap:
            name: nginx-config
      initContainers:
        - name: init-container
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - echo "<h1>OTUS HomeWORK 1</h1>" > /init/index.html;
          volumeMounts:
            - name: shared-volume
              mountPath: /init
      containers:
        - name: web-server
          image: nginx
          ports:
            - containerPort: 8000
          volumeMounts:
            - name: shared-volume
              mountPath: /homework
            - name: config-volume
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
          readinessProbe:  # Проверяем наличие файла /homework/index.html
            exec:
              command: ["/bin/sh", "-c", "test -f /homework/index.html"]
            initialDelaySeconds: 5
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "rm -f /homework/index.html"]

```
Ставим метку на ноду

```
kubectl label node node-1 homework=true
```


Заметка про расширение Deployment

Should you manually scale a Deployment, example via kubectl scale deployment deployment --replicas=X, and then you update that Deployment based on a manifest (for example: by running kubectl apply -f deployment.yaml), 
 then applying that manifest overwrites the manual scaling that you previously did. 


</details>

3. **HomeWork 3**

Изменить readiness-пробу в манифесте deployment.yaml из прошлого ДЗ на httpGet, вызывающую URL /index.html 
Необходимо создать манифест service.yaml, описывающий сервис типа ClusterIP, который будет направлять трафик на поды, управляемые вашим deployment
Установить в кластер ingress-контроллер nginx
Создать манифест ingress.yaml, в котором будет описан объект типа ingress, направляющий все http запросы к хосту homework.otus на ранее созданный сервис. В результате запрос http://homework.otus/index.html должен отдавать код html страницы, находящейся в подах

<details>
  <summary>Ответ</summary>

Устанавливаем  ingress-nginx и проверяем
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/baremetal/deploy.yaml
kubectl get pods -n ingress-nginx
```

Создаём ingress

```
kind: Ingress #Создаём объект типа ingress, который управляет входящими http/https запросами и направляет их на сервисы.
metadata:
  name: homework-ingress
  namespace: homework
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: / #Данная аннотация означает, что если в запросе будет /index.html то ingress перепишет его на /
spec:
  ingressClassName: nginx
  rules:
  - host: homework.otus
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: homework-service
              port:
                number: 80
```

Правим Service на Cluster ip

```
apiVersion: v1
kind: Service
metadata:
  name: homework-service
  namespace: homework
spec:
  selector:
    app: homework
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: ClusterIP
```

Правим тип проверки в deployment

```
          readinessProbe:
            httpGet: #проверка готовности будет выполняться http запросом
              path: /index.html # Страница которую мы запрашиваем для проверки
              port: 8000    # Порт на который выполняется http запрос, должен совпадать с containerPort:
              initialDelaySeconds: 5 # Ждём 5 сек после старта контейнера, прежде чем начать выполнять проверки
            periodSeconds: 10 # Проверка запускается каждые 10 сек.
          lifecycle:
            preStop: #Хук который выполняется перед остановкой контейнера
              exec:
                command: ["/bin/sh", "-c", "rm -f /homework/index.html"]

```
Запускаем

```
kubectl apply -f deployment.yaml
kubectl apply -f ingress.yaml
kubectl apply -f service.yaml
kubectl apply -f namespace.yaml

```

Проверка по DNS внутри кластера:

```
root@master-1:~/otustest/homework3# kubectl get svc -n homework
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
homework-service   ClusterIP   10.233.10.204   <none>        80/TCP    8d
```
```
curl http://homework-service.homework.svc.cluster.local/index.html
<h1>OTUS HomeWORK 3</h1>
root@master-1:~/otustest/homework3# curl http://10.233.10.204/index.html
<h1>OTUS HomeWORK 3</h1>
```
Проверка через тестовый контейнер


```
kubectl run curl-test --rm -it --image=alpine -n homework -- /bin/sh
apk add curl
# curl -H "Host: homework.otus" http://10.233.10.204/index.html
<h1>OTUS HomeWORK 3</h1>
```

</details>

4. **HomeWork 4** 

 


<details>
  <summary>Ответ</summary>

* Создать манифест pvc.yaml, описывающий PersistentVolumeClaim, запрашивающий хранилище с storageClass по-умолчанию.

* Создать манифест cm.yaml для объекта типа configMap, описывающий произвольный набор пар ключ-значени.

* В манифесте deployment.yaml изменить спецификацию volume типа emptyDir, который монтируется в init и основной контейнер, на pvc, созданный в предыдущем пункте.

* В манифесте deployment.yaml добавить монтирование ранее созданного configMap как volume к основному контейнеру пода в директорию /homework/conf, так, чтобы его содержимое можно было получить, обратившись по url /conf/file.

* Создать манифест storageClass.yaml описывающий объект типа storageClass с provisioner https://k8s.io/minikube-hostpath и reclaimPolicy Retain

Создаём pvc.yaml и Storageclass, после чего их применяем. Результатом будет создание PV.


```
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: homework-pvc
  namespace: homework
spec:
  storageClassName: "hw" 
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi  # Запрашиваем 100Mi
```


```
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hw
provisioner: k8s.io/minikube-hostpath  # Используем локальное хранилище
reclaimPolicy: Retain # После удаления PVC PV останется
volumeBindingMode: Immediate # Сразу привязываем PV и PVC
```

Меняем в deployment volume вместо emptydir на pvc.
```
initContainers:
        - name: init-container
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - echo "<h1>OTUS HomeWORK 3</h1>" > /init/index.html;
          volumeMounts:
            - name: homework-pvc
              mountPath: /init
      containers:
        - name: web-server
          image: nginx
          ports:
            - containerPort: 8000
          volumeMounts:
            - name: homework-pvc
              mountPath: /homework
            - name: config-volume
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf  # Монтируем только nginx.conf из configMap
            - name: homework-config
              mountPath: /homework/conf  # Монтируем файлы из cm.yaml в /homework/conf
```

Создаём манифест cm.yaml, с двумя парами ключ-значение.

```
# cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: homework-config
  namespace: homework
data:
  file1.txt: |
    OPOP 111
  file2.txt: |
    OPOP 2222
```

Проверка после применения всх манифестов:

```
kubectl run test --rm -it --image=alpine -- sh
add apk curl
curl http://homework-service:80/conf/file1.txt
curl http://homework-service:80/conf/file1.txt
```

```
kubectl port-forward svc/homework-service 8000:80 -n homework
curl http://localhost:8000/conf/file1.txt
curl http://localhost:8000/conf/file2.txt

```

</details>


5. **HomeWork 5**

В namespace homework создать service account monitoring и дать ему доступ к эндпоинту /metrics вашего кластера.
Изменить манифест deployment из прошлых ДЗ так, чтобы поды запускались под service account monitoring. 
В namespace homework создать service account с именем cd и дать ему роль admin в рамках namespace homework.
Создать kubeconfig для service account cd.
Сгенерировать для service account cd токен с временем действия 1 день и сохранить его в файл token/

<details>
  <summary>Ответ</summary>

В namespace homework создать service account monitoring и дать ему доступ к эндпоинту /metrics вашего кластера.
Изменить манифест deployment из прошлых ДЗ так, чтобы поды запускались под service account monitoring.   

Создаём манифест сервисного аккаунта
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring
  namespace: homework
secrets:
  - name: monitoring-service-account-token
```

Применяем:
```
kubectl apply -f service-account-monitoring.yaml
```


Создаём токен для аккаунта мониторинг
```
kubectl create token monitoring --duration=48h -n homework
```

Преобразуем в base64:

```
echo -n "$TOKEN" | base64
```

Создаём секрет для monitoring:

```
apiVersion: v1
kind: Secret
metadata:
  name: monitoring-service-account-token
  namespace: homework
  annotations:
    kubernetes.io/service-account.name: monitoring
type: kubernetes.io/service-account-token
data:
  token: |
    ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkltUXdXbkZsYVRWdWVIaFVkVVZ3TlRCamJIZGtl
    ...
```

Применяем:
```
kubectl apply -f secret-service-account-monitoring.yaml
```

Изменяем манифест deployment.yaml чтобы под запускались из под аккаунта monitoring
```
spec:
      serviceAccountName: monitoring  # Подключаем сервисный аккаунт monitoring
      nodeSelector:
```
Применяем:
```
kubectl apply -f deployment.yaml
```

Создаём Кластерную роль monitoring c соответствующими правами:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-role
rules:
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
```

Применяем:
```
kubectl apply -f role-monitoring.yaml
```

Связываем кластерную роль с сервисным аккаунтом monitoring:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-role
subjects:
  - kind: ServiceAccount
    name: monitoring
    namespace: homework
roleRef:
  kind: ClusterRole
  name: monitoring-role
  apiGroup: rbac.authorization.k8s.io
```

Применяем:
```
kubectl apply -f rolebinding-monitoring.yaml
```

Проверяем что метрики под данным аккаунтом можно получить и сервисы запущены под учеткой monitoring
```
curl -k \
  -H "Authorization: Bearer $TOKEN" \
  https://172.17.60.3:10250/metrics
```
![](images/metrics.png)

Проверяем что сервисы запущены под учеткой monitoring

```
kubectl get pods -n homework -o jsonpath="{range .items[*]}{.metadata.name}{'\t'}{.spec.serviceAccountName}{'\n'}{end}"
```
![](images/services.png)



В namespace homework создать service account с именем cd и дать ему роль admin в рамках namespace homework.
Создать kubeconfig для service account cd.
Сгенерировать для service account cd токен с временем действия 1 день и сохранить его в файл token/

Создаём сервисный аккаунт:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cd
  namespace: homework
```

Применяем:

```
kubectl apply -f service-account-cd.yaml
```

Связываем аккаунт cd c ролью

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cd-admin-binding
  namespace: homework
subjects:
  - kind: ServiceAccount
    name: cd
    namespace: homework
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

Применяем:

```
kubectl apply -f rolebinding-cd-admin.yaml
```

Создаём токен и сохраняем в файл:

```
kubectl create token cd --duration=24h -n homework > cd-token.txt
```

Получаем API сервера:
```
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'

```

Получаем рутовый сертификат:
```
kubectl get configmap kube-root-ca.crt -n homework -o jsonpath="{.data.ca\.crt}" > ca.crt
```

Присваиваем значения переменным и создаём cd-kubeconfig.yaml

```
export SERVER=https://127.0.0.1:6443
export TOKEN=$(cat cd-token.txt)

cat <<EOF > cd-kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: ca.crt
    server: ${SERVER}
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: homework
    user: cd
  name: cd-context
current-context: cd-context
users:
- name: cd
  user:
    token: ${TOKEN}
EOF
```

Проверяем 

```
KUBECONFIG=cd-kubeconfig.yaml kubectl get pods
```
![](images/kubeconfig.png)

</details>



8. **HomeWork 8**

Необходимо создать кастомный образ nginx, отдающий свои метрики на определенном endpoint 
● Установить в кластер Prometheus-operator любым удобным вам способом (рекомендуется ставить или по ссылке из офф документации, либо через helm-чарт) 
● Создать deployment запускающий ваш кастомный nginx образ и service для него 
● Настроить запуск nginx prometheus exporter (отдельным подом или в составе пода с nginx – не принципиально) и сконфигурировать его для сбора метрик с nginx 
● Создать манифест serviceMonitor, описывающий сбор метрик с подов, которые вы создали.

<details>
  <summary>Ответ</summary>


Устанавливаем оператор Prometheus через helm.


```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```



Скачиваем/устанавливаем/проверяем последнюю версию nerdctl
```
wget https://github.com/containerd/nerdctl/releases/download/v2.0.4/nerdctl-2.0.4-freebsd-amd64.tar.gz
tar -xzf nerdctl-2.0.4-freebsd-amd64.tar.gz
nerdctl --version
```


Скачиваем/устанавливаем/проверяем buildkit
```
curl -LO https://github.com/moby/buildkit/releases/download/v0.12.5/buildkit-v0.12.5.linux-amd64.tar.gz
tar -C /usr/local/bin -xzf buildkit-v0.12.5.linux-amd64.tar.gz
cp /tmp/buildkit/bin/* /usr/local/bin/
chmod +x /usr/local/bin/buildkitd /usr/local/bin/buildctl
which buildkitd
```



```
events {}

http {
    server {
        listen 80;
        location / {
            return 200 'OK';
            add_header Content-Type text/plain;
        }

        location /metrics {                   # отдаём stub_status нужный для экспорта метрик nginx-prometheus-exporter'ом
            stub_status;
        }
    }
}
```




делаем простой докер с кастомным конфигом nginx

```
FROM nginx:stable

RUN apt update && apt install -y curl && rm -rf /var/lib/apt/lists/*

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
```



собираем докер образ 
```
nerdctl build -t nginx-metrics:latest .
```


копируем образ на worker ноды
```
nerdctl save nginx-metrics:latest -o nginx-metrics.tar
scp nginx-metrics.tar node-1:/root/
scp nginx-metrics.tar node-2:/root/
```

Собираю deployment с кастомным образом nginx


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-metrics
  labels:
    app: nginx-metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-metrics
  template:
    metadata:
      labels:
        app: nginx-metrics
    spec:
      containers:
        - name: nginx
          image: nginx-metrics:latest   # наш локальный кастомно собранный образ
          imagePullPolicy: IfNotPresent # не скачиваем образ, т.к. он уже залит на ноды
          ports:
            - containerPort: 80

```

Описываем service для nginx
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-metrics
spec:
  selector:
    app: nginx-metrics
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

```
Собираем Deployment для экспортера

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-exporter
  template:
    metadata:
      labels:
        app: nginx-exporter
    spec:
      containers:
        - name: exporter
          image: nginx/nginx-prometheus-exporter:latest
          args:
            - -nginx.scrape-uri=http://nginx-metrics:80/stub_status
          ports:
            - containerPort: 9113
```

Создаём Servicemonitor который используется  операратором Prometheus для автоматического обнаружения и сбора метрик

```
apiVersion: monitoring.coreos.com/v1   # CRD который создал оператор Prometheus
kind: ServiceMonitor
metadata:
  name: nginx-exporter-monitor
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: nginx-exporter
  namespaceSelector:
    matchNames:
      - default
  endpoints:
    - port: http
      interval: 15s
```

kube-prometheus-stack - это release name, Helm автоматически добавляет этот label во все связанные объекты

Проверяем что это так:
```
 kubectl get prometheus -n monitoring -o yaml | grep release
      meta.helm.sh/release-name: kube-prometheus-stack
      meta.helm.sh/release-namespace: monitoring
      release: kube-prometheus-stack
        release: kube-prometheus-stack
        release: kube-prometheus-stack
        release: kube-prometheus-stack
        release: kube-prometheus-stack
        release: kube-prometheus-stack
```











Запускаем services и daemonsetы
```
kubectl apply -f nginx-metrics-deployment.yaml
kubectl apply -f nginx-metrics-service.yaml
kubectl apply -f nginx-exporter-deployment.yaml
kubectl apply -f nginx-exporter-service.yaml
kubectl apply -f nginx-servicemonitor.yaml
```


включаем форвардинг
```
kubectl port-forward svc/nginx-exporter 9113:9113
```


Проверяем, что метрики приходят:

```
root@master-2:~# curl http://localhost:9113/metrics
# HELP go_gc_duration_seconds A summary of the wall-time pause (stop-the-world) duration in garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
# HELP go_gc_gogc_percent Heap size target percentage configured by the user, otherwise 100. This value is set by the GOGC environment variable, and the runtime/debug.SetGCPercent function. Sourced from /gc/gogc:percent
# TYPE go_gc_gogc_percent gauge
go_gc_gogc_percent 100
# HELP go_gc_gomemlimit_bytes Go runtime memory limit configured by the user, otherwise math.MaxInt64. This value is set by the GOMEMLIMIT environment variable, and the runtime/debug.SetMemoryLimit function. Sourced from /gc/gomemlimit:bytes
# TYPE go_gc_gomemlimit_bytes gauge
go_gc_gomemlimit_bytes 9.223372036854776e+18
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 11
# HELP go_info Information about the Go environment.
# TYPE go_info gauge
go_info{version="go1.23.4"} 1
```
</details>