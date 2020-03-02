## Section4. Advanced Topics







### 4-1. DNS
- 클러스터의 모든 서비스에는 DNS 네임이 할당된다.
  - 예를 들어 foo라는 네임스페이스에 bar라는 pod가 있는 경우,
  - 다른 네임스페이스 quux에서는 bar라는 pod에 접근하기 위해서,
  - foo.bar를 조회하는 DNS 쿼리를 통해서 찾을 수 있다.
- kubernetes 1.3부터 DNS는 built-in 서비스로, addon manager에서 자동으로 실행된다.
- addons 정보는 master 노드의 /etc/kubernetes/addons 디렉토리에 존재한다  
- https://kubernetes.io/ko/docs/concepts/services-networking/dns-pod-service/
#### 서비스
##### A 레코드
- Normal(not headless) 서비스는 "my-svc.my-namespace.svc.cluster-domain.example" 형식의 DNS A 레코드 할당
  - 이는 클러스터 IP로 해석됨.
- Headless 서비스(클러스터 IP 없는) 역시 동일하게 "my-svc.my-namespace.svc.cluster-domain.example" 형식의 DNS A 레코드 할당되지만,
  - 이는 이 서비스로 선택된 파드들의 IP집합으로 해석 (이 IP집합에서 IP를 직접 선택 또는 Roundrobin 방식으로 선택)

#### Code Examples
- service-discovery/secrets.yml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: helloworld-secrets
type: Opaque
data:
  username: aGVsbG93b3JsZA==
  password: cGFzc3dvcmQ=
  rootPassword: cm9vdHBhc3N3b3Jk  #(base64로 디코딩하면 rootpassword)
  database: aGVsbG93b3JsZA==
```
- service-discovery/database.yml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database
  labels:
    app: database
spec:
  containers:
  - name: mysql
    image: mysql:5.7
    ports:
    - name: mysql-port
      containerPort: 3306
    env:
      - name: MYSQL_ROOT_PASSWORD  # secret에서 저장한 정보 활용
        valueFrom:
          secretKeyRef:
            name: helloworld-secrets
            key: rootPassword
      - name: MYSQL_USER
        valueFrom:
          secretKeyRef:
            name: helloworld-secrets
            key: username
      - name: MYSQL_PASSWORD
        valueFrom:
          secretKeyRef:
            name: helloworld-secrets
            key: password
      - name: MYSQL_DATABASE
        valueFrom:
          secretKeyRef:
            name: helloworld-secrets
            key: database
```
- database-service.yml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-service
spec:
  ports:
  - port: 3306
    protocol: TCP
  selector:
    app: database
  type: NodePort # NodePort 로 외부로 공개

```

- helloworld-db.yaml (DB에 접속하는 web server 살행 )
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: helloworld-db
  template:
    metadata:
      labels:
        app: helloworld-db
    spec:
      containers:
      - name: k8s-demo
        image: wardviaene/k8s-demo
        command: ["node", "index-db.js"]
        ports:
        - name: nodejs-port
          containerPort: 3000
        env:
          - name: MYSQL_HOST
            value: database-service # 이 hostname은 "database-service.yml"에서 미리 DNS에 등록한 호스트 명으로 접속 가능
          - name: MYSQL_USER
            value: root
          - name: MYSQL_PASSWORD
            valueFrom:
              secretKeyRef:
                name: helloworld-secrets
                key: rootPassword
          - name: MYSQL_DATABASE
            valueFrom:
              secretKeyRef:
                name: helloworld-secrets
                key: database
```
-helloworld-db-service.yml
  - 위에서 생성한 포드를 외부에 접속할 수 있는 서비스 (NodePort 오픈))
```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld-db-service
spec:
  ports:
  - port: 3000
    protocol: TCP
  selector:
    app: helloworld-db
  type: NodePort
```




```
> kubectl create -f service-discovery/secrets.yml
kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-s2jbq   kubernetes.io/service-account-token   3      30m
helloworld-secrets    Opaque                                4      32s

> kubectl create -f service-discovery/database.yml
pod/database created
> kubectl create -f service-discovery/database-service.yml
service/database-service created

> create -f service-discovery/helloworld-db.yml
deployment.apps/helloworld-deployment created

# minikube 실행시
> kubectl create -f service-discovery/helloworld-db-service.yml
service/helloworld-db-service created

# cloud 실행시
> kubectl expose deployment helloworld-deployment --type=LoadBalancer --name=helloworld-my-service

# DB 접속 확인
> kubectl logs helloworld-deployment-5464ff8c48-58rdw
Example app listening at http://:::3000
Connection to db established


# 서비스 동작 확인
> kubectl get services
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE
database-service        NodePort       10.15.247.250   <none>         3306:30268/TCP   24m
helloworld-db-service   NodePort       10.15.254.110   <none>         3000:32663/TCP   19m
helloworld-my-service   LoadBalancer   10.15.252.246   35.225.3.163   3000:32765/TCP   48s
kubernetes              ClusterIP      10.15.240.1     <none>         443/TCP          59m

> curl http://35.225.3.163:3000
Hello World! You are visitor number 1 (1이라는 숫자를 DB에 저장하여 조회)
> curl http://35.225.3.163:3000
Hello World! You are visitor number 2

# DB에 접속하여 확인하기
> kubectl exec database -it -- mysql -u root -p
Enter password: rootpassword #secret에 저장된 정보)
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| helloworld         |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use helloworld;

mysql> show tables;
+----------------------+
| Tables_in_helloworld |
+----------------------+
| visits               |
+----------------------+
1 row in set (0.00 sec)

mysql> select * from visits;
+----+---------------+
| id | ts            |
+----+---------------+
|  1 | 1582246156051 |
|  2 | 1582246219761 |
+----+---------------+
2 rows in set (0.00 sec)


# 생성한 서비스가 DNS에 잘 등록 되었는지 확인
# - 실제 강의와 다르게 DNS 명이 정상적으로 표시되지 않는다 (아마도 k8s 버전 차이인듯)
> kubectl run -i --tty busybox --image=busybox --restart=Never -- sh
If you dont see a command prompt, try pressing enter.
/ # nslookup helloworld-db-service
Server:		10.15.240.10
Address:	10.15.240.10:53 강의에서는 kube-dns.kube-system.svc.cluster.local이 여기 출력됨


Name:	helloworld-db-service.default.svc.cluster.local
Address: 10.15.254.110 강의에서는 위 Name이 여기 출력됨
```

### 4-2. COnfigMap
- key - value 쌍으로 구성 (volume 사용하여 mount)
- web server conf 파일처럼 전체 설정 파일도 활용 가능
- container의 이미지 변경없이 configuration을 통해 container 구성을 변경하는 경우 활용


#### Code Examples
- configmap/reverseproxy.conf
- url /로의 요청을  "http://127.0.0.1:3000"로 리다이렉트 하는 설정
```
server {
    listen       80;
    server_name  localhost;

    location / {
        proxy_bind 127.0.0.1;
        proxy_pass http://127.0.0.1:3000;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: helloworld-nginx
  labels:
    app: helloworld-nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.11
    ports:
    - containerPort: 80
    volumeMounts:
    - name: config-volume   # 아래 Volumes에 정의된 volume 지정
      mountPath: /etc/nginx/conf.d # 정한 volume 데이터를 pod의 이 경로로 mount
  - name: k8s-demo
    image: wardviaene/k8s-demo
    ports:
    - containerPort: 3000
  volumes:
    - name: config-volume
      configMap:
        name: nginx-config
        items:
        - key: reverseproxy.conf
          path: reverseproxy.conf
```

```shell
> kubectl create configmap nginx-config --from-file=configmap/reverseproxy.conf
configmap/nginx-config created
> kubectl get configmap
NAME           DATA   AGE
nginx-config   1      48s
> kubectl get configmap nginx-config -o yaml
apiVersion: v1
data:
  reverseproxy.conf: |    # <- 이 파일명을 나중에 pod에서 활용할 때 사용
    server {
        listen       80;
        server_name  localhost;

        location / {
            proxy_bind 127.0.0.1;
            proxy_pass http://127.0.0.1:3000;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2020-02-21T01:12:50Z"
  name: nginx-config
  namespace: default
  resourceVersion: "20126"
  selfLink: /api/v1/namespaces/default/configmaps/nginx-config
  uid: 85fbcb13-33ce-42fb-a142-13377cac9f87


> kubectl create -f configmap/nginx.yml
pod/helloworld-nginx created
>kubectl create -f configmap/nginx-service.yml
service/helloworld-nginx-service created

# 생성된 pod에 config 파일이 정상적으로 mount 되었는지 확인
> kubectl exec -it helloworld-nginx -c nginx -- bash
root@helloworld-nginx:/# cat /etc/nginx/conf.d/reverseproxy.conf
server {
    listen       80;
    server_name  localhost;

    location / {
        proxy_bind 127.0.0.1;
        proxy_pass http://127.0.0.1:3000;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

### 4-3. Ingress
- 클러스터로 들어오는 Inbound 연결을 허용
- 인그레스는 클러스터 외부에서 클러스터 내부 서비스로 HTTP와 HTTPS 경로를 노출한다.
- 트래픽 라우팅은 인그레스 리소스에 정의된 규칙에 의해 컨트롤된다.
  - 실제 동작을 위해서는 ingress controller가 필요하다.
  - 다양한 ingress controller가 있으며, 선택해야함
  - 선택된 ingress controller가 ingress에 정의된 rule에 따라 처리하는 방식
- HTTP, HTTPS 이외의 포트는 기존( external Loadbalancer와 NodePorts) 사용
  - Ingress는 service를 쉽게 외부에서 접속할 수 있도록 지원
- 기본으로 제공하는 Ingres를 사용하거나, 자체 제작한 ingress를 사용할 수 있다.
- https://kubernetes.github.io/ingress-nginx/deploy/  참고


### 4-4. External DNS
- public cloud에서는 LoadBalancer 가격을 줄이기 위해서 ingress controller를 사용할 수 있다.
  - 외부의 모든 요청을 받는 LoadBalancer에서 받아서 내부 ingress controller로 전달
  - 이 방식은 HTTPS, HTTP 기반의 어플리케이션 에서만 적용 가능
- 이를 가능하는 하는 도구가 "External DNS"이며
- 자동으로 필요한 DNS record를 외부 DNS Server(route53 같은)에 생성한다.
  - 즉, ingress에서 사용하는 모든 호스트명에 대해서 새로운 record를 생성
- 주요 DNS provider : Google CloudDNS, Route53, AzureDNS, CloudFlar, DIgitalOcean 등

#### Code Examples
- AWS를 사용한 실습 코드


### 4-5. volumes
- Container 외부 저장소에 데이터를 저장할 수 있는 기능
  - 컨테이너 중지(Stop)시 데이터 유실되는 상황을 방지하기 위함
- Persistent Volume
  - kubernetes에서는 container가 중지되어도 데이터를 volume에 저장하고,
  - container 재시작시 해당 volume을 container에 연결하는 기능 제공
- Volume plugins
  - AWS EBS, Google Disk, Azure Disk
  - Network Storage(NFS, Cephfs), Local Volume
#### StorageClass
- Volume은 성능에 따라 다양한 디스크 타입(HDD, SSD, NFS...)을 지정할 수 있는데,
- Kubernetes에서는 StorageClass라는 객체를 통해서 이를 정의한다 (V1.17에서는 Beta 버전)
- StorageClass의 특징은
  - Provision : 동적으로 자원(디스크)을 할당/배포할 수 있도록 지원
  - provision 지원 plugins : public cloud와 다양한 저장장치 지원
    - https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner
#### PersistentVolumeClaim
- 생성할 volume에 대한 접급권한 및 사이즈 등을 설정 가능

#### Volume을 pod에서 사용하는 방식
- 1. StorageClass를 정의 : 어떤 plugin을 사용해서 volume을 동적으로 할당 할 것인지 결정
- 2. PersistentVolumeClaim 정의
  - StorageClass에 정의된 서비스를 이용해서 volume을 생성하는데,
  - volume의 접근권한 및 사이즈를 정의한다.
- 3. Pod 생성
  - PersistentVolumeClaim을 pod의 특정 디렉토리에 mount한다.


#### Code Examples
- AWS에서 volume을 생성하고, 해당 볼륨의 ID를 확인한다.
```shell
> aws ec2 create-volume --size 10 --region your-region --availability-zone your-zone --volume-type gp2 --tag-specifications 'ResourceType=volume, Tags=[{Key= KubernetesCluster, Value=kubernetes.domain.tld}]'
```

- volumes/helloworld-with-volume.yml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
      - name: k8s-demo
        image: wardviaene/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
        volumeMounts:
        - mountPath: /myvol
          name: myvolume
      volumes:
      - name: myvolume
        awsElasticBlockStore:
          volumeID: # insert AWS EBS volumeID here
```

### 4-6. Pod Presets
- Pod Presets는 동작중인 pod에 데이터/정보를 삽입할 수 있도록 지원한다.
  - 예를 들어 secret, configmap, volume, environment 같은 자원을 pod에 적용
  - 운영 중인 pod가 많을 때, 이에 대한 설정을 한번에 쉽게 적용하는 용도로 사용 가능
  - Pod preset을 통해 pod에 자원을 삽입하면, pod 내의 모든 container에 적용됨.
- V1.17에서 alpha 버전임.
- 여러 개의 podpreset을 대상 pod에 적용할 수 있다. (만약 podpreset간의 충돌이 발생하면 적용 안됨)

#### Code Examples
  -
```yaml
apiVersion: settings.k8s.io/v1alpha1   # you might have to change this after PodPresets become stable
kind: PodPreset
metadata:
  name: share-credential
spec:
  selector:       #PodPreset을 적용할 pod를 지정한다.
    matchLabels:
      app: myapp
  env:                # env, volume 등의 정보를 해당 pod에 적용한다.
    - name: MY_SECRET
      value: "123456"
  volumeMounts:
    - mountPath: /share
      name: share-volume
  volumes:
    - name: share-volume
      emptyDir: {}
```

### 4-7. StatefulSets
- StatefulSets는 상태 유지가 필요한 어플리케이션(DB 등) 용도로 도입되었다.
- 기본 조건으로 pod의 hostname이 정적으로 할당되어야 한다.
  - pod가 재생성 될때 마다 동적으로 바뀌면 안됨.
  - 이를 위해서 DNS를 활용하도록 한다.
  - 예를 들어 podname-0, podname-1 등과 같이 "hostname-index"의 형태로 관리
- 따라서 StatefulSet이 삭제 또는 중지되는 경우에도 Volume의 데이터가 유지되어야 한다.
- 또한 StatefulSet은 pod의 시작/중시 순서를 지정할 수 있도록 한다. (무작위로 선택하지 않고)
  - 예를 들어 pod의 scale up할 때 node의 index가 0에서 n-1까지 증가하도록 설정
  - Scale down은 반대로..

#### Code Examples
- statefulset/casandra.yaml 참고


```shell

# StorageClas에서 cloud사용시 regoin을 instance node의 region과 동일하도록 변경해야 함
> kubectl create -f cassandra.yaml
statefulset.apps/cassandra created
storageclass.storage.k8s.io/standard created
service/cassandra created

# pod를 확인해보면, replica가 3개인데, 1개만 먼저 생성되고 있다.
# statefulset은 하나의 pod다 정상적으로 생성되어야 다음 pod가 생성되는 방식
> kubectl get pod
NAME          READY   STATUS    RESTARTS   AGE
NAME          READY   STATUS              RESTARTS   AGE
cassandra-0   1/1     Running             0          3m1s
cassandra-1   1/1     Running             0          108s
cassandra-2   0/1     ContainerCreating   0          17s


> kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                STORAGECLASS   REASON   AGE
pvc-0c97c54c-5e65-4a33-bdfd-9735fe9023d0   8Gi        RWO            Delete           Bound    default/cassandra-data-cassandra-1   standard                2m28s
pvc-2b001292-a3ed-4f2a-8aee-7544692aebf3   8Gi        RWO            Delete           Bound    default/cassandra-data-cassandra-0   standard                3m20s
pvc-c787060c-1a58-429e-9fb9-bb2ede9b9d4a   8Gi        RWO            Delete           Bound    default/cassandra-data-cassandra-2   standard                58s

> kubectl get pvc
NAME                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cassandra-data-cassandra-0   Bound    pvc-2b001292-a3ed-4f2a-8aee-7544692aebf3   8Gi        RWO            standard       9m14s
cassandra-data-cassandra-1   Bound    pvc-0c97c54c-5e65-4a33-bdfd-9735fe9023d0   8Gi        RWO            standard       2m22s
cassandra-data-cassandra-2   Bound    pvc-c787060c-1a58-429e-9fb9-bb2ede9b9d4a   8Gi        RWO            standard       51s


# 아래 명령어로 3개의 casandra pod 정보 확인
# 각 pod는 서로 연결되어 통신이 가능함.
> kubectl exec -it cassandra-0 -- nodetool status
Datacenter: DC1-K8Demo
======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  100.96.3.3  65.87 KiB  32           66.2%             f91c3489-8175-44af-adfc-ca4c8713408e  Rack1-K8Demo
UN  100.96.1.3  70.91 KiB  32           71.7%             26935384-3627-44bb-9dcf-0b36038f2d0a  Rack1-K8Demo
UN  100.96.2.3  104.57 KiB  32           62.1%             df813740-d47f-4415-b610-9c4201215606  Rack1-K8Demo


# pod간 통신이 되는지 ping으로 확인해 보자
# cassandra-0에서 cassandra-1로 연결이 정상적으로 됨.
kubectl exec -it cassandra-0 -- bash
root@cassandra-0:/# ping cassandra-1.cassandra
PING cassandra-1.cassandra.default.svc.cluster.local (100.96.1.3): 56 data bytes
64 bytes from 100.96.1.3: icmp_seq=0 ttl=62 time=0.359 ms
64 bytes from 100.96.1.3: icmp_seq=1 ttl=62 time=0.338 ms

# statefulset pod인 cassandra-2를 삭제해보자.
> kubectl delete pod cassandra-2
pod "cassandra-2" deleted

# pod는 바로 재시작 된다. (26초 전에 생# )
> kubectl get pod
kNAME          READY   STATUS    RESTARTS   AGE
cassandra-0   1/1     Running   0          10m
cassandra-1   1/1     Running   0          9m17s
cassandra-2   0/1     Running   0          26s

# PV(볼륨)을 보면 해당 볼륨은 삭제되지 않고, 그대로 유지가 됨.
# 이를 통해서 데이터의 유실을 막고, 상태를 유지함.
> kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                STORAGECLASS   REASON   AGE
pvc-0c97c54c-5e65-4a33-bdfd-9735fe9023d0   8Gi        RWO            Delete           Bound    default/cassandra-data-cassandra-1   standard                9m18s
pvc-2b001292-a3ed-4f2a-8aee-7544692aebf3   8Gi        RWO            Delete           Bound    default/cassandra-data-cassandra-0   standard                10m
pvc-c787060c-1a58-429e-9fb9-bb2ede9b9d4a   8Gi        RWO            Delete           Bound    default/cassandra-data-cassandra-2   standard                7m48s
```

### 4-8. DaemonSets
- 특정 pod를 모든 노드에 1개씩 구동하는 경우 활용
- 만약 노드가 삭제도면, 해당 pod는 다른 노드로 재배치 되지 않는다. (노드당 1개만 실행되므로)
- 활용 예시
  - 로그 수집기(에이전트)
  - 모니터링 에이전트
  - 로드밸런서/API Gateway/Reverse Proxy등


### 4-9. Resource Usage monitoring
- Heapster는 container cluster의 모니터링과 성능 분석이 가능한 툴이다.
- Heapster가 없으면 kubernetes에서 pod의 auto scaling이 불가능하다.
- Rest API를 통해서 cluster의 metric을 전달함.
- Heapster를 통한 모니터링에 필요한 도구들 (모두 pod로 실행 가능)
  - InfluxDB : 수집한 데이터를 저장하는 backend능
  - Grafana : 데이터 시각화
#### 내부 프로세스
- 각 노드의 kubelet(내부에 cAdvisor 프로세스)이 각 pod의 정보를 수집함
- kubelet은 Heapster pod에 모니터링 정보를 전송하고,
- Heapster는 해당 정보를 InfluxDB저장한 후, Grafana로 시각화 한다.
#### Heapster --> Metric Server 전환
- Heapster project는 더이상 진행되지 않고
- Metric-Server로 전환되어서 진행된다.

#### Code Examples
- https://github.com/kubernetes-sigs/metrics-server 코드 활용
- 사전 조건으로 kops로 aws에 k8s cluster를 생성한다.
- https://blog.naver.com/freepsw/221758787689
```shell
> kubectl create -f metrics-server
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
serviceaccount/metrics-server created
deployment.apps/metrics-server created
service/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created

# 모니터링 메트릭이 수집도는지 확인 (초기 서비스 배포에 시간 소요)
> kubectl top node
error: metrics not available yet
# 3분정도 후 메트릭 정보 수집 확인
> kubectl top node
NAME                                               CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-172-20-36-133.ap-northeast-2.compute.internal   20m          2%     374Mi           41%
ip-172-20-49-153.ap-northeast-2.compute.internal   80m          8%     682Mi           76%
ip-172-20-53-32.ap-northeast-2.compute.internal    108m         10%    353Mi           39%


# 테스트용 pod 배포
> kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.4 --port=8080
deployment.apps/hello-minikube created


# pod 모니터링 지표 확인
> kubectl top pod
NAME                              CPU(cores)   MEMORY(bytes)
hello-minikube-75cb6dd856-xv7tk   0m           2Mi

```


### 4-10. AutoScaling
- 모티터링된 지표를 기반으로 deployment, replication controller, replica set을 확장 가능
- k8s 1.3에서 cpu기반의 auto scaling 가능
  - alpha 버전으로 초당 요청건수 및 평균 지연시간에 따른 auto scaling 지원
- Cluster의 아래 환경설정(environment variable)을 변경하여 cluster를 시작해야 함.
  - ENABLE_CUSTOM_METRICS = true
#### metric 확인 주기
- 기본 30초
- Controller-manager 실행시 "--horizontal-pod-autoscaler-sync-period"를 이용하여 변경 가능
#### 예시
- CPU 자원이 200m인 pod 배포
  - 200m = 200 milicpu (or 200 milicores)
  - 200m = 0.2(현재 node의 CPU core에서 20%를 의미 )
    - node의 core가 2개 있더라도, 한개 core의 20%를 의미
- Auto scaling 기준을 pod의 CPU 사용률 50%로 정하면,
  - 실제 node 기준으로는 100m(현재 pod가 200m 할당 받았고, 여기서 50%는 100m)


#### Code Examples
- autoscaling/hpa-example.yaml
- CPU 200m 할당 받은 pod 정의
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-example
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hpa-example
  template:
    metadata:
      labels:
        app: hpa-example
    spec:
      containers:
      - name: hpa-example
        image: gcr.io/google_containers/hpa-example
        ports:
        - name: http-port
          containerPort: 80
        resources:
          requests:
            cpu: 200m
```

- hpa-example deployment의 자원 사용률이 50%(200m의 50% = 100m) 초고하면
- pod를 최대 10개까지 확장
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-example-autoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-example
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

```shell
> kubectl create -f autoscaling/hpa-example.yml
deployment.apps/hpa-example created
service/hpa-example created
horizontalpodautoscaler.autoscaling/hpa-example-autoscaler created

> kubectl get hpa
NAME                     REFERENCE                TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
hpa-example-autoscaler   Deployment/hpa-example   <unknown>/50%   1         10        3          30s

# busybox pod를 생성해서, hpa-example deployment에 정상 접속 되는지 확인
> run -i --tty load-generator --image=busybox /bin/sh
/ # wget http://hpa-example.default.svc.cluster.local:31001
Connecting to hpa-example.default.svc.cluster.local:31001 (100.67.118.244:31001)
saving to 'index.html'
index.html           100% |*******************************************************************************************************|     3  0:00:00 ETA
'index.html' saved

# hpa-example pod에 부하를 주고, auto scaling 작동을 확인하자.
> while true; do wget -q -O- hpa-example.default.svc.cluster.local:31001; done
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!


# hpa에서 자원 사용률을 확인하면, 138%까지 증가했음이 보인다.
# 이는 50% 기준으로 3개의 pod가 필요
> kubectl get hpa
NAME                     REFERENCE                TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
hpa-example-autoscaler   Deployment/hpa-example   138%/50%   1         10        1          7m14s

# 138%의 부하를 유지하기 위해 3개 pod까지 확장 됨
> kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
hpa-example-688ddd5567-82vh9      1/1     Running   0          7m11s
hpa-example-688ddd5567-fj4km      1/1     Running   0          7s
hpa-example-688ddd5567-tfkl7      1/1     Running   0          7s
load-generator-5fb4fb465b-vvnsl   1/1     Running   0          5m24s
```
- 이 상태에서 부하를 중지시키면, 원래 pod 크기 만큼으로 줄어든다

### 4-11. Affinity & Anti-affinity
- 이전에 nodeSelector(특정 label을 지정한 노드를 선택하는 옵션)을 배웠었다.
- Affinity는 더 복잡한 스케줄링이 필요할때 사용하는 방법 (Ndoe와 pod 모두 적용 가능 )
- Node affinity : nodeSelector와 유사
- Pod(Interpod) affinity
  - 다른 실행중인 pod를 고려하여, 현재 pod를 어떻게 스케줄링 할지 룰을 설정 가능
- Affinity rule은 pod가 스케줄링 되는 동안에만 적용 가능
  - pod가 실행되고 나면, affinity를 변경해도 적용되지 않음. (재시작 해야 함)
#### Node Affinity (2가지 유형)
- requiredDuringSchedulingIgnoredDuringExecution
  - 엄격한 룰 적용 필요 (nodeSelector 유사)
  - 룰에 맞지 않으면, pod를 스케줄링 할 수 없음.
  ```yaml
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      - key: env
        operator: In
        values:
        - dev
  ```
- preferredDuringSchedulingIgnoredDuringExecution
  - 룰을 먼저 확인하지만, 룰에 맞지 않아도 스케줄링 가 (룰에 맞으면 적용하고, 아니면 말고)
  ```yaml
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 1
    preference:
      matchExpressions:
      - key: team
        operator: In
        values:
        - engineering-project1
  ```
  - weight는 룰에 맞는 노드가 여러개 있을때 가중치를 통해 선택하는 용도
  - 예를 들어 2개의 룰이 있고, 각 룰의 weight가 1점, 5점인 경우
  - 어떤 노드는 2개 룰을 모두 충족하여 weight 6이 되고,
  - 어떤 노드는 1개 룰만 만족하여 1점이 될때,
  - 스케줄링 시에 6점이 노드에 배포되도록 우선순위 정의 가능
#### Built in Node label
- https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#built-in-node-labels
- Kubernetes에서 node 추가시 자동으로 생성되는 라벨 (스케줄링시 참고)
```
kubernetes.io/hostname
failure-domain.beta.kubernetes.io/zone
failure-domain.beta.kubernetes.io/region
topology.kubernetes.io/zone
topology.kubernetes.io/region
beta.kubernetes.io/instance-type
node.kubernetes.io/instance-type
kubernetes.io/os
kubernetes.io/arch
```

#### Code Examples

```shell
> kubectl get nodes
NAME                                               STATUS   ROLES    AGE   VERSION
ip-172-20-36-133.ap-northeast-2.compute.internal   Ready    node     81m   v1.16.7
ip-172-20-49-153.ap-northeast-2.compute.internal   Ready    master   83m   v1.16.7
ip-172-20-53-32.ap-northeast-2.compute.internal    Ready    node     81m   v1.16.7

# 노드의 라벨을 확인해 보자 (자동으로 생성된 라벨 확인 가능 )
> kubectl describe node ip-172-20-36-133.ap-northeast-2.compute.internal
Name:               ip-172-20-36-133.ap-northeast-2.compute.internal
Roles:              node
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=t2.micro
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=ap-northeast-2
                    failure-domain.beta.kubernetes.io/zone=ap-northeast-2a
                    kops.k8s.io/instancegroup=nodes
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-172-20-36-133.ap-northeast-2.compute.internal
                    kubernetes.io/os=linux
                    kubernetes.io/role=node
                    node-role.kubernetes.io/node=
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true


# affinityf를 적용한 deployment를 실행했지만, 스케줄링이 안되고 pending 상태 유지  
> kubectl create -f affinity/node-affinity.yaml
deployment.apps/node-affinity created
skiper@skiperui-MacBook-Pro kubernetes-course % kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
node-affinity-7cbcc5f4f-m2cg5   0/1     Pending   0          4s
node-affinity-7cbcc5f4f-sdswx   0/1     Pending   0          4s
node-affinity-7cbcc5f4f-wnx2b   0/1     Pending   0          4s

# requiredDuringSchedulingIgnoredDuringExecution 룰을 만족하지 못하므로,
# node에 label을 추가해 보자.
> kubectl label node ip-172-20-36-133.ap-northeast-2.compute.internal env=dev

> kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
node-affinity-7cbcc5f4f-m2cg5   1/1     Running   0          4m22s
node-affinity-7cbcc5f4f-sdswx   1/1     Running   0          4m22s
node-affinity-7cbcc5f4f-wnx2b   1/1     Running   0          4m22s
```

### 4-12. Interpod Affinity and anti-affinity
- 이미 실행중인 pod를 기준으로 현재 pod가 배포되도록 스케줄링 하는 방식
- 활용 사례
  - application과 redis cache pod가 항상 같은 node에 존재
#### topologyKey
- pod의 affinity의 룰을 지정할때 topologyKey 사용 (node label과 동일한 의미)
- 만약 affinity 룰이 충족되면, 새로운 pod는 동일한 topologyKey(현재 실행중인 pod와 동일한)를 가진 node에 배포된다.
- 사용 가능한 연산자
  - In, NotIn, Exists, DoesNotExist

### 예시를 통한 설명
- 기존에 실행 중인 pod 정보
- pod의 label이 'app=pod-affinity-1'
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-affinity-1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pod-affinity-1
  template:
    metadata:
      labels:
        app: pod-affinity-1
    spec:
      containers:
      - name: k8s-demo
        image: wardviaene/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
```

- Interpod Affinity를 이용한 pod 실행
- pod-affinity-1 pod와 동일한 node에서 실행되도록 룰을 정의
- AntiAffinity는 이를 반대로 적용하면 됨. (즉, 룰에 매칭된 노드에서 실행되지 않게)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-affinity-2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pod-affinity-2
  template:
    metadata:
      labels:
        app: pod-affinity-2
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions: # 1. pod-affinity-1 label을 가진 pod를 찾고
                  - key: "app"
                    operator: In
                    values:
                    - pod-affinity-1
              topologyKey: "kubernetes.io/hostname"  # 2. 해당 pod와 동일한 host(node)에서 실행
      containers:
      - name: redis
        image: redis
        ports:
        - name: redis-port
          containerPort: 6379
```

### 4-13. Taints and Tolerations
- Node에 Taint로 특정 마크(표시)를 하고,
- Pod에서는 toleration로 pod 스케줄링시에 taint를 참고하여 규칙에 맞는 노드 배포한다.
- Tolerations (pod를 기준으로 규칙 설정)
  - node affinity와 반대되는 개념
  - 특정 pod들을 지정된 node에서 실행되지 못하도록 설정
- 활용 예시
  - 특정 팀/조직에서만 활용하는 노드로 지정 또는 GPU서버를 특정 권한을 가진 사람에게 지정
  - 또는 pod를 생성할 때 master node에는 배포되지 않도록 설정
  - 이를 위해 Master Node는 taint가 마크되어야 함.
    - node-role.kubernetes.io/master:NoSchedule
    - 명령어 예시 : kubectl taint node1 key=value:NoSchedule
  - Toleration을 이용하여 taint된 node1에 새로운 pod가 스케줄링 되도록 설정
    ```yaml
    spec:
      tolerations:
      - key: "type"
        operator: "Equal" # "Exists 는 key가 있는지만 확인하는 operator"
        value: "value"
        effect: "NoSchedule"
    ```
#### Taint Effect
- NoSchedule : 엄격한 룰 적용. 룰에 맞지 않으면 노드에 배포 불가
- PreferNoSchedule : 가능하면 toleration에 맞는 pod를 배포 (필수조건은 아님 )
- NoExecute : 현재 실행중인 pod에 Taint 적용 (신규 적용된 taint에 맞지 않는 pod를 다른 노드로 이동-evict, 종료 후 다른 노드에서 시작)
  - NoSchedule은 pod가 배포되는 시점에서만 적용. (신규 taint를 적용해도 반영 안됨. )
  - 여기서 pod 이동시 tolerationTime 활용 (Pod를 언제 이동 할지 결정)
    - pod 이동시 tolerationTime만큼 기다렸다가 종료하고 다른 노드에서 실행
    - tolerationTime이 없는 경우 바로 종료하고 다른 노드에서 실행


#### Examples Code

```shell
# kops로 생성한 k8s node를 확인해보고,
# 이중 master node에 어떤 taint가 적용되었는지 보자.
> kubectl get nodes
NAME                                               STATUS   ROLES    AGE    VERSION
ip-172-20-36-133.ap-northeast-2.compute.internal   Ready    node     174m   v1.16.7
ip-172-20-49-153.ap-northeast-2.compute.internal   Ready    master   176m   v1.16.7  # master node
ip-172-20-53-32.ap-northeast-2.compute.internal    Ready    node     174m   v1.16.7

> kubectl get nodes ip-172-20-49-153.ap-northeast-2.compute.internal -o yaml |less
..........
spec:
  taints:  # NoSchedule로 적용되어, toleraton이 없는 pod는 생성되지 못함.
  - effect: NoSchedule
    key: node-role.kubernetes.io/master

```


- k8s node에 taint를 생성하고,
- toleration을 적용한 pod를 생성해 보자.
```shell
> kubectl taint nodes ip-172-20-36-133.ap-northeast-2.compute.internal type=specialnode:NoSchedule
node/ip-172-20-36-133.ap-northeast-2.compute.internal tainted
```
- 아래 deployment 실행
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tolerations-1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tolerations-1
  template:
    metadata:
      labels:
        app: tolerations-1
    spec:
      containers:
      - name: k8s-demo
        image: wardviaene/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tolerations-2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tolerations-2
  template:
    metadata:
      labels:
        app: tolerations-2
    spec:
      tolerations:
      - key: "type"  # type=specialnode 확인
        operator: "Equal"
        value: "specialnode"
        effect: "NoSchedule"
      containers:
      - name: k8s-demo
        image: wardviaene/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
```

```shell
> kubectl create -f tolerations/tolerations.yaml
deployment.apps/tolerations-1 created
deployment.apps/tolerations-2 created

> kubectl get nodes
NAME                                               STATUS   ROLES    AGE    VERSION
ip-172-20-36-133.ap-northeast-2.compute.internal   Ready    node     3h6m   v1.16.7
ip-172-20-49-153.ap-northeast-2.compute.internal   Ready    master   3h8m   v1.16.7
ip-172-20-53-32.ap-northeast-2.compute.internal    Ready    node     3h6m   v1.16.7


# 현재 taint가 설정된 node는 master와 ip-172-20-36-133.ap-northeast-2.compute.internal 노드 2개
# tolerations-1 pod는 별도의 toleration이 없으므로, taint된 node에는 배포 불가 --> 마지막 노드(ip-172-20-53-32.ap-northeast-2.compute.internal)에만 배포
# tolerations-2는 ip-172-20-36-133.ap-northeast-2.compute.internal의 taint와 일치함.
# 따라서 ip-172-20-36-133.ap-northeast-2.compute.internal노드와 ip-172-20-53-32.ap-northeast-2.compute.internal 2군데 배포가 가능함.
>  kubectl get pod -o wide
NAME                             READY   STATUS    RESTARTS   AGE    IP            NODE                                               NOMINATED NODE   READINESS GATES
tolerations-1-684d45c7df-567qx   1/1     Running   0          5m1s   100.96.2.12   ip-172-20-53-32.ap-northeast-2.compute.internal    <none>           <none>
tolerations-1-684d45c7df-6rh2q   1/1     Running   0          5m1s   100.96.2.11   ip-172-20-53-32.ap-northeast-2.compute.internal    <none>           <none>
tolerations-1-684d45c7df-bhcg5   1/1     Running   0          5m1s   100.96.2.9    ip-172-20-53-32.ap-northeast-2.compute.internal    <none>           <none>
tolerations-2-786dfd8fc8-5xtlb   1/1     Running   0          5m1s   100.96.1.12   ip-172-20-36-133.ap-northeast-2.compute.internal   <none>           <none>
tolerations-2-786dfd8fc8-9zjwz   1/1     Running   0          5m1s   100.96.1.13   ip-172-20-36-133.ap-northeast-2.compute.internal   <none>           <none>
tolerations-2-786dfd8fc8-r4thz   1/1     Running   0          5m1s   100.96.2.10   ip-172-20-53-32.ap-northeast-2.compute.internal    <none>           <none>

# taint 삭제
> kubectl taint nodes ip-172-20-53-32.ap-northeast-2.compute.internal testkey-
node/ip-172-20-53-32.ap-northeast-2.compute.internal untainted
> kubectl taint nodes ip-172-20-36-133.ap-northeast-2.compute.internal type2-
node/ip-172-20-36-133.ap-northeast-2.compute.internal untainted
> kubectl taint nodes ip-172-20-36-133.ap-northeast-2.compute.internal type-
node/ip-172-20-36-133.ap-northeast-2.compute.internal untainted
```

### 4-14. Custom Resource Definition (CRD)
- K8s API를 확장할 수 있는 도구
- K8s에서 리소스란 API object를 저장한 K8s API 내부 ednpoint들의 집합이다.
  - 예를 들어 K8s는 기본으로 제공하는 deployment라는 자원이 있으며, 이를 통해 pod를 배포한다.
  - yaml에서는 deployment라는 자원을 이용하여 object를 정의할 수 있다.
  - kubectl 명령어를 통해서 클러스터 상의 object 생성도 가능
- 자신의 클러스터에서 특별한 용도로 CRD를 생성하여, 별도의 기능을 추가할 수 있다.
- Operator 역시 CRD를 활용하여 생성한다.


### 4-15. Operators
- K8s 어플리케이션의 패키징, 배포, 관리를 위한 도구(method)
  - https://coreos.com/operators
- Operator는 개별 어플리케이션에 운영방법(지식)을 포함하도록 도와준다.
  - 사용자는 배포하는 모든 application을 전부 알 필요 없이,
  - Cloud의 managed service와 같이 사용할 수 있다. (쉽게 배포/운영 가능)
  - Opoerators가 배포되고 나면, Custom Resource Definition를 통해서 관리 가능
- Operators를 통해서 복잡한 stateful service를 k8에 배포 가능
  - 복잡한 내부 구조는 CRD를 통해 추상화하여, 쉽게 사용 가능
- 다양한 3-party operator 제작 가능
  - Prometheus, Vault, Rook, MySQL, PostgreSQL 등
  - PostgreSQL Operators를 생성하는 경우
    - replica, failover, backup, scale 등이 가능한 기능을 제공해야 함.
    - Operator는 위와 같은 기능들 다양한 management logic을 통해
    - 관리지/사용자의 직접 구현 없이 사용할 수 있게 해 준다.

#### Code Examples
- https://github.com/CrunchyData/postgres-operator 참고
- PostgreSQL Operator Creates/Configures/Manages PostgreSQL Clusters on Kubernetes

```shell
> cd postgres-operator
> ls
README.md		quickstart-for-gke.sh	set-path.sh		storage.yml
> cat storage.yml 
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  zone: ap-northeast-2a


```
