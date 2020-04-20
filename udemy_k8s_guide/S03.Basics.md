## Section 3. Kubernetes Basics

### 3-1. Replication controller
#### Code Examples
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: helloworld-controller
spec:
  replicas: 2
  selector:
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
```

```shell
> kubectl create -f replication-controller/helloworld-repl-controller.yml

>  kubectl get pods

> kubectl delete pod helloworld-controller-gbqmb

> kubectl scale --replicas=4 -f replication-controller/helloworld-repl-controller.yml
NAME                              READY   STATUS    RESTARTS   AGE
helloworld-controller-2zqbm       1/1     Running   0          8m5s
helloworld-controller-g4w5p       1/1     Running   0          5m17s
helloworld-controller-k2vnb       1/1     Running   0          5m17s
helloworld-controller-n5qg6       1/1     Running   0          7m5s

> kubectl get rc
NAME                    DESIRED   CURRENT   READY   AGE
helloworld-controller   4         4         4       11m
> kubectl scale --replicas=1 rc/helloworld-controller
replicationcontroller/helloworld-controller scaled
> kubectl get pods
NAME                              READY   STATUS        RESTARTS   AGE
helloworld-controller-2zqbm       1/1     Running       0          9m26s
helloworld-controller-g4w5p       1/1     Terminating   0          6m38s
helloworld-controller-k2vnb       1/1     Terminating   0          6m38s
helloworld-controller-n5qg6       1/1     Terminating   0          8m26s

> kubectl delete rc/helloworld-controller
replicationcontroller "helloworld-controller" deleted
>kubectl get rc
No resources found in default namespace.
```


### 3-2. Deployments
#### Replication Set
- Replication Controller의 진화된 다음 버전
- 새로운 selector를 지원하여 값에 따라 필터링하여 selection이 가능함
  - 예를 들면 "environment" 값이 "dev" 또는 "qa"로 2개의 조건이 가능함
- Deployment에서는 Replication Set을 사용함. (RC는 사용하지 않음)
- 즉, 처음부터 Replication Set을 쓰면 되는것 아닌가?
#### Deployments
- 어플리케이션의 배포 및 업데이트 용도로 사용됨
- Deployment를 사용하면, 어플리케이션의 상태(state)를 정의할 수 있다.
- 그럼 왜 Replication Contorller/Replication Set을 직접 쓰지 않을까?
  - 위 방식은 앱 베포를 위해서 개발자가 해야 할 작업이 많다. (update/rollback 등)
- 어떻게 편하게 할 수 있나? (아래와 같이 단순한 명령어로 처리 가능)
  - create deployment
  - update deployment
  - rolling updates (zero downtime)
  - roll back (이전 버전으로 복구)
  - pause / resume a deployment (특정 퍼센트만 roll-out 가능함)


#### Code Examples
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 3
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
```

```shell
# Depolyments를 생성하면, Replication Set까지 자동으로 생성해 준다.

> kubectl create -f deployment/helloworld.yml
deployment.apps/helloworld-deployment created
> kubectl get deployments
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-deployment   1/3     3            1           7s
>kubectl get rs  <-- Replication Set 생성 확인
NAME                              DESIRED   CURRENT   READY   AGE
helloworld-deployment-d6fc545f7   3         3         3       38s
>kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
helloworld-deployment-d6fc545f7-bwtkp   1/1     Running   0          50s
helloworld-deployment-d6fc545f7-g5zf2   1/1     Running   0          50s
helloworld-deployment-d6fc545f7-hzdg7   1/1     Running   0          50s
> kubectl get pods --show-labels
NAME                                    READY   STATUS    RESTARTS   AGE     LABELS
helloworld-deployment-d6fc545f7-bwtkp   1/1     Running   0          3m14s   app=helloworld,pod-template-hash=d6fc545f7
helloworld-deployment-d6fc545f7-g5zf2   1/1     Running   0          3m14s   app=helloworld,pod-template-hash=d6fc545f7
helloworld-deployment-d6fc545f7-hzdg7   1/1     Running   0          3m14s   app=helloworld,pod-template-hash=d6fc545f7

# 상태 확인
>kubectl rollout status deployment/helloworld-deployment
deployment "helloworld-deployment" successfully rolled out

# 외부로 서비스 오픈
>kubectl expose deployment helloworld-deployment --type=NodePort
service/helloworld-deployment exposed
>kubectl get service
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
helloworld-deployment    NodePort    10.96.44.63     <none>        3000:32593/TCP   14s
kubernetes              ClusterIP   10.96.0.1     <none>        443/TCP          83s
>kubectl describe service helloworld-deployment
Name:                     helloworld-deployment
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=helloworld
Type:                     NodePort
IP:                       10.96.44.63
Port:                     <unset>  3000/TCP
TargetPort:               3000/TCP
NodePort:                 <unset>  32593/TCP
Endpoints:                172.17.0.2:3000,172.17.0.6:3000,172.17.0.7:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

# 접속 가능한 URL 확인 (minikube 제공)
>minikube service helloworld-deployment --url
http://192.168.99.101:32593
>curl http://192.168.99.101:32593
Hello World!


# 이전 버전(version 2)로 롤백해 보자
>kubectl set image deployment/helloworld-deployment k8s-demo=wardviaene/k8s-demo:2
deployment.apps/helloworld-deployment image updated
>kubectl rollout status deployment/helloworld-deployment
Waiting for deployment "helloworld-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "helloworld-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "helloworld-deployment" successfully rolled out
>curl http://192.168.99.101:32593
Hello World v2!
>kubectl get pods
NAME                                    READY   STATUS        RESTARTS   AGE
helloworld-deployment-bc54cf79c-qjtjc   1/1     Running       0          23s
helloworld-deployment-bc54cf79c-twm76   1/1     Running       0          35s
helloworld-deployment-bc54cf79c-xk6qw   1/1     Running       0          28s
helloworld-deployment-d6fc545f7-bwtkp   1/1     Terminating   0          12m
helloworld-deployment-d6fc545f7-g5zf2   1/1     Terminating   0          12m
helloworld-deployment-d6fc545f7-hzdg7   1/1     Terminating   0          12m

>kubectl rollout history deployment/helloworld-deployment
deployment.apps/helloworld-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

# 버전 2로 업데으트 한 것을 다시 원래대로 복구(uno)해 보자
>kubectl rollout undo deployment/helloworld-deployment
deployment.apps/helloworld-deployment rolled back
>kubectl rollout status deployment/helloworld-deployment
Waiting for deployment "helloworld-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "helloworld-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "helloworld-deployment" successfully rolled out
>kubectl get pods
NAME                                    READY   STATUS        RESTARTS   AGE
helloworld-deployment-bc54cf79c-qjtjc   1/1     Terminating   0          2m19s
helloworld-deployment-bc54cf79c-twm76   1/1     Terminating   0          2m31s
helloworld-deployment-bc54cf79c-xk6qw   1/1     Terminating   0          2m24s
helloworld-deployment-d6fc545f7-69hn5   1/1     Running       0          18s
helloworld-deployment-d6fc545f7-7hchw   1/1     Running       0          12s
helloworld-deployment-d6fc545f7-hxt69   1/1     Running       0          7s
>curl http://192.168.99.101:32593
Hello World!

# 아래와 같이 히스토리가 2개만 보이는 것은 설정에서 변경 가능
# revisisionHistoryLimit 설정 변경 (기본 2)
>kubectl rollout history deployment/helloworld-deployment
deployment.apps/helloworld-deployment
REVISION  CHANGE-CAUSE
2         <none>
3         <none>

# revision 히스토리를 기준으로 특정 시점으로 undo 가능
> kubectl rollout undo deployment/helloworld-deployment --to-revision=2

# 생성한 deployments 삭제
>kubectl delete deployment/helloworld-deployment
```

### 3-3. Services
- 어플리케이션을 배포하면, pod가 동적으로 생성/삭제 되는데,
- 이때 외부에서 pod에 직접 접속하게 되면, IP가 계속 변경되는 문제가 있다.
- 이를 위해 외부에서 접속할 수 있게 제공하는 서비스 (LoadBalance와 같은 역할)
  - ClusterIP : 클러스터 내부 IP (클러스터 내부에서만 접속 가능)
  - NodePort : 노드(서버)의 포트를 제공 (외부에서 접속 가능), 30000 ~ 32767 구간만 오픈 가능(kube-apiserver에서 설정으로 변경 가능 --service-node-port-range=...)
  - LoadBalancer : 퍼블릭 클라우드 서비스를 사용해야 활용 가능 (외부 접속을 모든 NodePort로 연결해 준다)

#### Code Examples
- pod를 생성하고, 이를 서비스로 오픈하는 예시
- pod 생성 스크립트
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodehelloworld.example.com
  labels:
    app: helloworld
spec:
  containers:
  - name: k8s-demo
    image: wardviaene/k8s-demo
    ports:
    - name: nodejs-port
      containerPort: 3000

```
- service 생성 스크립트
```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld-service
spec:
  ports:
  - port: 31001
    nodePort: 31001
    targetPort: nodejs-port
    protocol: TCP
  selector:
    app: helloworld
  type: NodePort
```

```shell
>kubectl create -f first-app/helloworld.yml
pod/nodehelloworld.example.com created
>kubectl get pods
NAME                         READY   STATUS              RESTARTS   AGE
nodehelloworld.example.com   0/1     ContainerCreating   0          4s
>kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
nodehelloworld.example.com   1/1     Running   0          11s

# 서비스를 오픈하고 접속해 보자.
>kubectl create -f first-app/helloworld-nodeport-service.yml
service/helloworld-service created
>minikube service helloworld-service --url
http://192.168.99.101:31001
>curl http://192.168.99.101:31001
Hello World!

>kubectl get svc
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
helloworld-service   NodePort    10.96.114.228   <none>        31001:31001/TCP   99s
kubernetes           ClusterIP   10.96.0.1       <none>        443/TCP           29m

# 사용한 자원 삭제
> kubectl delete service helloworld-service
> kubectl delete pods nodehelloworld.example.com
```

### 3-4. LABELS
- Key/Value 쌍으로 구성 (tag와 같은 역할)
#### Node Labels
- pod가 특정 label을 가진 node에서만 실행될 수 있도록 설정 가능
  - Node에 label을 설정
    ```shell
    > kubectl label nodes node1 hardware=high-spec
    > kubectl label nodes node2 hardware=low-spec
    > kubectl get nodes --show-labels
    ```
  - Pod 설정에서 NodeSelector에 해당 label을 지정
    ```yaml
    spec:
      nodeSelector:
        hardware:high-spec
    ```

### 3-5. Health Checks
- pod로 배포한 어플리케이션이 잘못 동작하고 있지만,
- 실제 pod가 중지된 상태가 아니라면, 사용자는 잘못된 서비스(에러 등)를 받게 될 것이다.
- 이러한 문제를 해결하기 위해서, health check을 통해서 주기적으로 어플리케이션의 상태를 확인한다.
  - container 내부에서 특정 command를 실행하여 확인
  - url을 호출하여 정상 결과값이 오는지 확인


#### Code Examples
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 3
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
        livenessProbe: # health check를 실행하는 코드
          httpGet:
            path: /
            port: nodejs-port
          initialDelaySeconds: 15 # 초기 시작시 15초간은 확인하지 않음
          timeoutSeconds: 30 # healthcheck 요청후 30초간 응답없으면, 장애로 판단
```

```shell
>kubectl create -f deployment/helloworld-healthcheck.yml
deployment.apps/helloworld-deployment created
>kubectl get deployments
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-deployment   1/3     3            1           7s
```

### 3-6. Readiness Probs
- livenessProbes
  - container가 실행중인지 확인하는 지표
  - 확인에 실패하면, container를 재시작 한다.
- ReadnessProbs
  - container가 정상적인 서비스를 할 수 있는 상태인지 체크
  - 확인에 실패하면, container를 삭제하지 않고,
  - 해당 container를 서비스에서 제외한다. (더 이상 서비스를 하지 못하도록)
  - 초기 container 구동시 테스트를 하여, 서비스에 포함할지 결정한다.
- 보통 2개의 prob를 동시에 활용하여 서비스/컨테이너 상태를 제어한다.

#### Code Examples
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-readiness
spec:
  replicas: 3
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
        livenessProbe:
          httpGet:
            path: /
            port: nodejs-port
          initialDelaySeconds: 15
          timeoutSeconds: 30
        readinessProbe:
          httpGet:
            path: / # livenessProbe와 동일한 경로를 체크 (이체 정상으로 체크되어야 container 실행 )
            port: nodejs-port
          initialDelaySeconds: 15
          timeoutSeconds: 30
```

```shell
# 그럼 기존 liveness로 실행한 container와 동작방식이 어떻게 다른가?
## 기존 liveness만 있는 container는 생성시 바로 running 상태로 전환
## readness가 추가되면서, readness의 체크가 끝난 후에 running 상태로 변경됨  
>kubectl create -f deployment/helloworld-liveness-readiness.yml
deployment.apps/helloworld-readiness created
>kubectl get pods
NAME                                     READY   STATUS              RESTARTS   AGE
helloworld-deployment-667857ccbf-8wz9t   1/1     Running             0          29m
helloworld-deployment-667857ccbf-tp9sz   1/1     Running             0          29m
helloworld-deployment-667857ccbf-xqvgl   1/1     Running             0          29m
helloworld-readiness-54747697b5-qz9xj    0/1     ContainerCreating   0          2s  <-- 이 부분에서 차이
helloworld-readiness-54747697b5-w2s7s    0/1     ContainerCreating   0          2s
helloworld-readiness-54747697b5-wrnmx    0/1     ContainerCreating   0          2s
```

### 3-7. Pod State
- Pod Status : 상위 레벨의 상태 제공(kubectl get pods)
- Pod Condition : pod의 조건(kubectl describe pod <pod_name>)
- Container Stat : Pod 내부의 실제 container의 상태
#### Pod State의 코드 유형
- Pending : Pod는 접수(정상적으로 실행요청)되었지만, 아직 실행 준비 중
  - container image를 다운로드 받는 상태
  - 필요한 자원을 할당받기 위해서 대기 중
- Succeeded : Pod 내의 모든 container가 성공적으로 종료되었고, 재시작하지 않음
- Failed : Pod 내의 모든 container가 종료되었지만, 1개 이상의 container가 Fail Code를 리턴
  - 보통 exit code를 리턴
- Unknown : 알수 없는 pod의 상태
  - network error로 인한 경우 (예를 들어 pod가 실행중인 노드가 중지되는 경우 )
#### Pod Condition의 코드 유형
- PodScheduled : pod가 노드에 할당됨
- Ready : pod가 요청을 받을 준비(readness)가 되었으며, Service에 등록될 예정임.
- Initialized : initialization container가 성공적으로 실행됨. -> pod lifecycle에서 다시 정리
- Unschedulable : 자원 제약등의 이유로 Pod가 할당되지 못함.

#### Container State 코드 유형
- Container(docker)의 내부 코드 (Running, Terminated, Waiting)

#### Pod Lifecycle
- Init Container
  - 새로운 container를 실행하면, 실제 동작할 main container와 별개로
  - 별도의 작업을 실행하게 된다. (예를 들면 볼륨설정시 디렉토리 생성 및 접근권한 설정 등)
  - 이런 작업은 main container와 완전히 별개로 실행되어야 함.
  - 이 작업이 정상적으로 끝나야만, main container 실행이 가능하다.
- Main Container
  - post start hook : main container 시작과 동시에 특정 작업(yaml에 정의한)을 실행
  - pre stop hook : main container 종료 전에 특정 작업을 실행
  - readness/liveness probs : initialDelaySeconds 후에 main container에 health check 실행 .


##### Code Examples
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: lifecycle
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lifecycle
  template:
    metadata:
      labels:
        app: lifecycle
    spec:
      initContainers: # main container 실행 전에 별도 container 실행(sleep 10)
      - name:           init
        image:          busybox
        command:       ['sh', '-c', 'sleep 10']
      containers:     # main container(실제 배포할 어플리케이션, echo로 현재 시간을 /timing에 저장하고, 120초 sleep)
      - name: lifecycle-container
        image: busybox
        command: ['sh', '-c', 'echo $(date +%s): Running >> /timing && echo "The app is running!" && /bin/sleep 120']
        readinessProbe: # 현재 시간을 /timing 파일에 기록할 수 있는지 체크  
          exec:
            command: ['sh', '-c', 'echo $(date +%s): readinessProbe >> /timing']
          initialDelaySeconds: 35
        livenessProbe:
          exec:
            command: ['sh', '-c', 'echo $(date +%s): livenessProbe >> /timing']
          initialDelaySeconds: 35
          timeoutSeconds: 30
        lifecycle:    # post start/pre stop 시점에 실행할 명령어 정의
          postStart:
            exec:
              command: ['sh', '-c', 'echo $(date +%s): postStart >> /timing && sleep 10 && echo $(date +%s): end postStart >> /timing']
          preStop:
            exec:
              command: ['sh', '-c', 'echo $(date +%s): preStop >> /timing && sleep 10']
```

```shell
>kubectl create -f pod-lifecycle/lifecycle.yaml
deployment.apps/lifecycle created
>kubectl get pods
NAME                         READY   STATUS     RESTARTS   AGE
lifecycle-69f486767f-4q9j6   0/1     Init:0/1   0          13s
>kubectl exec -it lifecycle-69f486767f-4q9j6 -- cat /timing
1580537014: Running
1580537014: postStart
1580537024: end postStart
1580537053: readinessProbe
1580537058: livenessProbe
1580537062: readinessProbe
1580537068: livenessProbe
1580537072: readinessProbe
1580537078: livenessProbe
1580537082: readinessProbe
```

### 3-8. Secrets
- 민감한 정보(패스워드, 개인정보 등)을 안전하게 관리할 수 있도록 K8s에서 제공하는 기능
- 어플리케이션에 민감정보를 전달하는 용도 (vault와 같은 도구도 활용 가능 )
- 적용 방식
  - environment 변수로 관리
  - file로 관리
    - 민감정보가 저장된 파이의 Volume을 container에 mount
    - dotenv 파일 활용 (어플리케이션에서 직접 접속)
  - 외부 이미지(private registry)를 활용하여 민감정보를 pull 방식으로 가져온다.  

#### Code Examples
- Secret 생성
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
type: Opaque
data:
  username: cm9vdA==
  password: cGFzc3dvcmQ=
```
- Secret(db-secrets)을 활용하여 Deployment내부 container의 volume 지정하기
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 3
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
        - name: cred-volume
          mountPath: /etc/creds
          readOnly: true
      volumes:
      - name: cred-volume
        secret:
          secretName: db-secrets
```

```shell
>cd DevTools/github/kubernetes_exercise/code/kubernetes-course/
>kubectl create -f deployment/helloworld-secrets.yml
secret/db-secrets created
>kubectl create -f deployment/helloworld-secrets-volumes.yml
deployment.apps/helloworld-deployment created
>kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
helloworld-deployment-6898b65669-4dfs7   1/1     Running   0          12s
helloworld-deployment-6898b65669-jp9wm   1/1     Running   0          12s
helloworld-deployment-6898b65669-tjf7l   1/1     Running   0          12s


# 생성된 pod의 정보 조회
>kubectl describe pod/helloworld-deployment-6898b65669-4dfs7
....
Mounts:
  /etc/creds from cred-volume (ro)
  /var/run/secrets/kubernetes.io/serviceaccount from default-token-5z6gj (ro)
....
Volumes:
  cred-volume:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  db-secrets
    Optional:    false
  default-token-5z6gj:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-5z6gj
    Optional:    false
....

# 생성된 secret 정보가 container에 저장되었는지 확인하기 위해 container 접속
>kubectl exec helloworld-deployment-6898b65669-4dfs7 -i -t -- /bin/bash

# 접속된 container에서 "/etc/creds" 디렉토리 내용 확인

root@helloworld-deployment-6898b65669-4dfs7:/app# cat /etc/creds/password
password

# 여기서 "password"라고 보이는 이유는 secret 생성 시점에 base64로 인코딩 한 내용이
# container에 저장되면서, 정상 인코딩으로 변환된것으로 보임
# "password"를 인코딩하면, echo "password" | base64
# cGFzc3dvcmQK 로 출력되는 것이 확인 가능
```

#### Code Examples (Running Wordpress on Kubernetes)
- wordpress secrets.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: wordpress-secrets
type: Opaque
data:
  db-password: cGFzc3dvcmQ=
```

- wordpress-single-deployment-no-volumes.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:4-php7.0
        ports:
        - name: http-port
          containerPort: 80
        env:
          - name: WORDPRESS_DB_PASSWORD # DB에 접속하기 위한 DB 패스워드 정의 (secret활용 )
            valueFrom:
              secretKeyRef:
                name: wordpress-secrets
                key: db-password
          - name: WORDPRESS_DB_HOST
            value: 127.0.0.1
      - name: mysql
        image: mysql:5.7
        ports:
        - name: mysql-port
          containerPort: 3306
        env:
          - name: MYSQL_ROOT_PASSWORD # MySQL root 패스워드 정의 (secret 활용 )
            valueFrom:
              secretKeyRef:
                name: wordpress-secrets
                key: db-password
```

```shell
>kubectl create -f wordpress/wordpress-secrets.yml
secret/wordpress-secrets created
>kubectl create -f wordpress/wordpress-single-deployment-no-volumes.yml
deployment.apps/wordpress-deployment created
>kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
wordpress-deployment-56868f9bf7-z9f8j   2/2     Running   1          3m8s

# 서비스 생성
>kubectl create -f wordpress/wordpress-service.yml
service/wordpress-service created
>kubectl get services
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
kubernetes          ClusterIP   10.96.0.1      <none>        443/TCP           25h
wordpress-service   NodePort    10.96.18.243   <none>        31001:31001/TCP   20s

# 외부에서 접속 가능한 url 확인
>minikube service wordpress-service --url
http://192.168.99.101:31001


# 자원 삭제
>kubectl get deployments
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
wordpress-deployment   1/1     1            1           6m22s
>kubectl delete deployment/wordpress-deployment
deployment.apps "wordpress-deployment" deleted
>kubectl get services
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
kubernetes          ClusterIP   10.96.0.1      <none>        443/TCP           25h
wordpress-service   NodePort    10.96.18.243   <none>        31001:31001/TCP   4m31s
>kubectl delete service/wordpress-service
service "wordpress-service" deleted
```

### 3-9. Web UI
- kubectl 명령어로 전체 자원을 확인하는 것은 불가능함
- 이를 web ui로 조회 할 수 있도록 지원
  - 기본 https://kubernetes-master-ip/ui 로 접속 가능
  - 접속이 어려운 경우 직접 설치 가능
    - kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
    - 패스워드는 아래 설정에서 확인 가능
      - kubectl config view
- minikube를 사용하는 경우
  - minikube dashboard 로 생성 가능
  - 접속 url은
    - minikube dashboard --url
      http://127.0.0.1:52017/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/

#### Kops에서 Web UI 실행하기

##### Start dashboard

Create dashboard:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

##### Create user
Create sample user (if using RBAC - on by default on new installs with kops / kubeadm):
```
kubectl create -f sample-user.yaml

```

##### Get login token:
```
kubectl -n kube-system get secret | grep admin-user
kubectl -n kube-system describe secret admin-user-token-<id displayed by previous command>
```

##### Login to dashboard
Go to http://api.yourdomain.com:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login

Login: admin
Password: the password that is listed in ~/.kube/config (open file in editor and look for "password: ..."




## ETC
### Port 구분
- NodePort : 외부에서 접속하기 위해 사용하는 포트
- port : Cluster 내부에서 사용할 Service 객체의 포트
- targetPort : Service객체로 전달된 요청을 Pod(deployment)로 전달할때 사용하는 포트

- 흐름으로 보면 NodePort --> Port --> targetPort  
