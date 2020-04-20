## Section5. Administration


### 5-1. Master Services
- kubrectl을 통해 rest i/f(Kube-api Server)로 새로운 오브젝트나 자원을 전달한다.
- 또한 전달된 모든 object 정보는 back end인 etcd에 저장된다.
  - 예를 들어 pod definition 과 같은.
- Scheduler
  - pod에 대한 자원을 할당하는 역할을 하며,
  - plugin 방식으로 다양한 scheduler를 사용할 수 있다.
- Controller Manager
  - 다수의 controller는 유형별로 다양하게 존재한다.
  - node controller는 새로운 node를 발견하거나, 관리하는 역할
  - replication controller는 pod의 복제본을 관리하는 역할
- kubelet
  - node에서 실행되며, Kube-api server와 통신한다. (node에서 실제 작업)

### 5-2. Resource Quotas
- 다수의 사용자/조직에게 자원을 할당하는 방법 정의
- k8s 클러스터 내에서 Namespace를 생성하고, 각 namespace 별로 자원을 할당  
  - ResourceQuota, ObjectQuota 로 구현 가능
####  ResourceQuota
- Request capacity : pod가 실행하는데 필요한 최소한의 자원 정의
- Resource limit : pod가 사용가능한 최대의 자원 정의 (그 이상은 할당 불가)
- Capacity Quota
  - Administrator가 정의하며, 모든 pod가 여기서 정해진 자원을 초과하여 사용할 수 없음
  - 사용자가 별도로 pod에 자원을 명시하지 않았다면, 기본으로 적용되는 quota (request, limit)
  - pod가 여기서 정의된 값 이상을 요청하는 경우, 403 error 발생
#### Object Quota
- administrator는 모든 object에 대해서도 제약을 설정 가능하다
- configmas 최대 개수, pod 최대 개수 등
https://kubernetes.io/docs/concepts/policy/resource-quotas/#object-count-quota 참고

### 5-3. Namespace
- 하나의 물리적 클러스터 내에서 가상의 클러스터 생성가능
- 즉, 물리적인 클러스터를 논리적으로 여려개로 분할하여 관리 가능
- 기본 namespace는 "default"
  - 사용자가 별도의 namespace를 만들지 않으면,
  - 기본으로 default namespace에 모든 자원을 생성하게 된다.
- kube-system namespace
  - K8s 관리에 필요한 자원을 관리하는 namespace
  - 모니터링, dns 등의 서비스는 여기서 실행됨
- Namespace 내부 자원의 이름 관리
  - Namespace 내부에서 모든 자원은 유일한 값을 가져야 함.
  - 다른 namespace와는 상관 없음 (중복 가능)
- Namespace 자원 할당
  - namespace 별로 최대 자원을 할당 가능 (CPU, Memory, loadbalancer 등)
  ```yaml
  kind: ResourceQuota
  metadata:
    name: object-quota    # Object quota 할당
    namespace: myspace    # namespace 지정
  spec:
    hard:
      configmaps: "10"
      persistentvolumeclaims: "4"
      replicationcontrollers: "20"
      secrets: "10"
      services: "10"
      services.loadbalancers: "2"
  ```


#### Code Examples
##### 1. Namespace를 생성하고, 자원(resource, object)를 할당
- resourcequotas/resourcequota.yml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myspace
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: myspace
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
  namespace: myspace
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
```

```shell
> kubectl create -f resourcequotas/resourcequota.yml
namespace/myspace created
resourcequota/compute-quota created
resourcequota/object-quota created
```

##### 2. quota를 지정하지 않은 deployment 실행
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
  namespace: myspace
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
> kubectl create -f resourcequotas/helloworld-no-quotas.yml
deployment.apps/helloworld-deployment created

# myspace에 배포된 deployment 확인
# 그런데 ready 상태에 있는 pod가 하나도 없다.
> kubectl get pod  --namespace=myspace
No resources found in myspace namespace.

> kubectl get deployment --namespace=myspace
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-deployment   0/3     0            0           29s  <-- 실행중인 pod가 없다.

> kubectl get rs --namespace=myspace
NAME                              DESIRED   CURRENT   READY   AGE
helloworld-deployment-d6fc545f7   3         0         0       79s

# 세부 상태를 확인하니, "failed quota" 에러가 보인다.
# 즉, quota를 지정하지 않아서, pod를 할당할 수 없는것.
> kubectl describe rs/helloworld-deployment-d6fc545f7 --namespace=myspace
.......
Events:
  Type     Reason        Age                  From                   Message
  ----     ------        ----                 ----                   -------
  Warning  FailedCreate  3m20s                replicaset-controller  Error creating: pods "helloworld-deployment-d6fc545f7-sbrnf" is forbidden: failed quota: compute-quota: must specify limits.cpu,limits.memory,requests.cpu,requests.memory
```
##### 3. quota를 지정한 새로운 deployment를 배포
- helloworld-with-quotas.yml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-deployment
  namespace: myspace
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
        resources:
          requests:
            cpu: 200m
            memory: 0.5Gi
          limits:
            cpu: 400m
            memory: 1Gi
```

```shell
# 기존 deployment 삭제
> kubectl delete deployment/helloworld-deployment --namespace=myspace
deployment.apps "helloworld-deployment" deleted
> kubectl create -f resourcequotas/helloworld-with-quotas.yml
deployment.apps/helloworld-deployment created

# 이번엔 running중인 pod가 2개만 있다. (replica 3개를 요청했는데.. )
# 아마도 namespace 자원이 부족해서 그런 것일 듯..
> kubectl get deployment --namespace=myspace
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-deployment   2/3     2            2           27s
> kubectl get pod --namespace=myspace
NAME                                     READY   STATUS    RESTARTS   AGE
helloworld-deployment-6dd55ffb5d-569gd   1/1     Running   0          2m28s
helloworld-deployment-6dd55ffb5d-pvqdg   1/1     Running   0          2m28s
> kubectl get rs --namespace=myspace
NAME                               DESIRED   CURRENT   READY   AGE
helloworld-deployment-6dd55ffb5d   3         2         2       2m47s

# 마지막 라인에서 "exceeded quota" 오류가 보인다.
> kubectl describe rs/helloworld-deployment-6dd55ffb5d --namespace=myspace
.........
Conditions:
  Type             Status  Reason
  ----             ------  ------
  ReplicaFailure   True    FailedCreate
Events:
  Type     Reason            Age                    From                   Message
  ----     ------            ----                   ----                   -------
  Normal   SuccessfulCreate  3m20s                  replicaset-controller  Created pod: helloworld-deployment-6dd55ffb5d-pvqdg
  Warning  FailedCreate      3m20s                  replicaset-controller  Error creating: pods "helloworld-deployment-6dd55ffb5d-dz545" is forbidden: exceeded quota: compute-quota, requested: limits.memory=1Gi,requests.memory=512Mi, used: limits.memory=2Gi,requests.memory=1Gi, limited: limits.memory=2Gi,requests.memory=1Gi
  Normal   SuccessfulCreate  3m20s                  replicaset-controller  Created pod: helloworld-deployment-6dd55ffb5d-569gd
  Warning  FailedCreate      3m20s                  replicaset-controller  Error creating: pods "helloworld-deployment-6dd55ffb5d-lqsfb" is forbidden: exceeded quota: compute-quota, requested: limits.memory=1Gi,requests.memory=512Mi, used: limits.memory=2Gi,requests.memory=1Gi, limited: limits.memory=2Gi,requests.memory=1Gi
  Warning  FailedCreate      3m20s                  replicaset-controller  Error creating: pods "helloworld-deployment-6dd55ffb5d-qq2tf" is forbidden: exceeded quota: compute-quota, requested: limits.memory=1Gi,requests.memory=512Mi, used: limits.memory=2Gi,requests.memory=1Gi, limited: limits.memory=2Gi,requests.memory=1Gi

# 아래 myspace의 자원할당(compute-quota)을 보면 최대 메모리가 2Gi 인데,
# 각 pod에서 1G씩 사용하고, 나머지 pod에 할당될 자원이 부족해서 할당을 못하는 현상을 확인
> kubectl get quota --namespace=myspace
NAME            CREATED AT
compute-quota   2020-03-03T00:45:52Z
object-quota    2020-03-03T00:45:52Z

>  kubectl describe quota/compute-quota --namespace=myspace
Name:            compute-quota
Namespace:       myspace
Resource         Used  Hard
--------         ----  ----
limits.cpu       800m  2
limits.memory    2Gi   2Gi
requests.cpu     400m  1
requests.memory  1Gi   1Gi
```

##### 4. pod생성시 default quota를 지정해 주자
- pod 생성시 별도 quota를 요청하지 않으면, namespace에서 디폴트 값으로 설정하도록 구성
- resourcequotas/default.yml
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
  namespace: myspace
spec:
  limits:
  - default:
      cpu: 200m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 256Mi
    type: Container
```

```shell
> kubectl create -f resourcequotas/defaults.yml
limitrange/limits created

> kubectl get limits --namespace=myspace
NAME     CREATED AT
limits   2020-03-03T09:35:46Z

> kubectl describe limits limits --namespace=myspace
Name:       limits
Namespace:  myspace
Type        Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---  ---------------  -------------  -----------------------
Container   memory    -    -    256Mi            512Mi          -
Container   cpu       -    -    100m             200m           -

# 이제 새로운 deployment를 배포해 보자. (quota 지정 )
> kubectl create -f resourcequotas/helloworld-no-quotas.yml
deployment.apps/helloworld-deployment created

> kubectl get deployment --namespace=myspace
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-deployment   3/3     3            3           7s
> kubectl get rs --namespace=myspace
NAME                              DESIRED   CURRENT   READY   AGE
helloworld-deployment-d6fc545f7   3         3         3       16s
> kubectl get pod --namespace=myspace
NAME                                    READY   STATUS    RESTARTS   AGE
helloworld-deployment-d6fc545f7-hpwp7   1/1     Running   0          21s
helloworld-deployment-d6fc545f7-vpp54   1/1     Running   0          21s
helloworld-deployment-d6fc545f7-xxv2c   1/1     Running   0          21s
```

### 5-4. User Management
#### 2가지 유형의 사용자
  - normal user : kubectl을 통해서 접근 가능한 사용자. object를 이용해서 관리할 수 없다.
  - service user : k8s의 object로 관리하는 사용자로, cluster 내에서 인증하는 용도로 사 (Secret과 비슷한 용도)
#### 사용자 인증 방식
- normal user
  - client 인증서를 통한 인증 : 보통 간단하게 이 방식으로 진행
  - Bearer token 방식
  - Authentication Proxy 방식 :
  - HTTP Basic Authentication : password 전송
  - OpenID : Cloud 사업자들이 제공하는 방식
- service user
  - Service Account Tokens : Secret에 인증정보를 저장하고, 이를 pod에서 활용할 수 있도록 pod의 디렉토리에 마운트 함
  - 인증되지 않은 API call은 anonymous 사용자로 처리
#### 사용자 Attribute (사용자 속성정보)
- normal user
  - username (e.g user123 or user@gmail.com)
  - uid
  - groups
  - extra field (다양한 속성을 저장하기 위한 필드 )
#### Authorization
- https://kubernetes.io/docs/reference/access-authn-authz/authorization/
- Authentication 이후 사용자는 허락된 서비스에만 접근할 수 있는 다양한 Authorization을 적용한다.
- AlwaysAllow / AlwaysDeny
- ABAC (Attribute-Based Access Control)  : 수동으로만 설정 가능
- RBAC (Role-Based Access Control) : rbac.authorization.k8s.io API group 활용 (API로 동적 변경 가능)
- Webhook (원격 서비스에 의해서 권한 부여 )

#### Code Examples
##### K8s 인증서
- OpenSSL 로 ROOT CA 생성 및 SSL 인증서 발급 (https://www.lesstif.com/system-admin/service-management/openssl-root-ca-ssl)
- k8s 인증서를 통한 통신 (https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-trust)
- PKI 인증서 및 요구 조건 (https://kubernetes.io/ko/docs/setup/best-practices/certificates/ )
- openssl can manually generate certificates for your cluster (https://kubernetes.io/docs/concepts/cluster-administration/certificates/)

##### 2) GCP GKE 환경에서 사용자 인증서 생성하기 (필요한가?)
```shell
# k8s server로 접속 (cloud vm을 사용하는 경우 ssh로 접속)
>  ssh -i ~/.ssh/gcp-freepsw freepsw@34.67.2xx.xxx

# 접속한 K8S node에서 ssh 인증을 위한 인증서 생성
# 1) 새로운 인증 키 생성  
> openssl genrsa -out gcp-freepsw-k8s.pem 2048
Generating RSA private key, 2048 bit long modulus
................................+++++
......+++++
e is 65537 (0x10001)

> ls
gcp-freepsw-k8s.pem

# 2) 인증서 요청(CSR-Certificate Signing Request) 생성
## CSR 은 인증기관에 인증서 발급 요청을 하는 특별한 ASN.1 형식의 파일이며(PKCS#10 - RFC2986)  
## 그 안에는 내 공개키 정보와 사용하는 알고리즘 정보등이 들어 있다. 개인키는 외부에 유출되면 안 되므로 이렇게 특별한 형식의 파일을 만들어서 인증기관에 전달하여 인증서를 발급 받는다.
> openssl req -new -key gcp-freepsw-k8s.pem -out gcp-freepsw-k8s-csr.pem -subj '/CN=gcp-freepsw-sk/O=mytem/'
> ls
gcp-freepsw-k8s-csr.pem  gcp-freepsw-k8s.pem


# 3) CSR을 이용하여 인증서(.crt) 생성
## 내가 생성할 인증서를 인증해줄 root CA(인증서)를 지정해야 함. (Minikube /var/lib/localkube/ca.crt)
## Minikube에서는 root CA를 자체적으로 생성하여 관리 (k8s cluster 내부에서 사용하는 인증서의 root CA로 활용)
> sudo openssl x509 -req -in gcp-freepsw-k8s-csr.pem -CA /var/lib/localkube
```

##### 2) Minikube 환경에서 사용자 인증서 생성하기
- Minikube는 자체적으로 root CA(ca.crt, ca.key)를 제공하므로, 이를 root CA로 지정하여 새로운 사용자 인증서 생성
###### 2-1) Minikube에서 사용자 인증서 생성하기.
```shell
# k8s server로 접속 (cloud vm을 사용하는 경우 ssh로 접속)
> ssh -i ~/.ssh/gcp-freepsw freepsw@34.67.2xx.xxx
> minikube ssh (아래 실습은  minikube 환경에서 실행)

# 1) 새로운 인증 키 생성  
> openssl genrsa -out mini-freepsw.pem 2048

# 2) 인증서 요청(CSR-Certificate Signing Request) 생성
## CSR 은 인증기관에 인증서 발급 요청을 하는 특별한 ASN.1 형식의 파일이며(PKCS#10 - RFC2986)  
## 그 안에는 내 공개키 정보와 사용하는 알고리즘 정보등이 들어 있다. 개인키는 외부에 유출되면 안 되므로 이렇게 특별한 형식의 파일을 만들어서 인증기관에 전달하여 인증서를 발급 받는다.
> openssl req -new -key mini-freepsw.pem -out mini-freepsw-csr.pem -subj "/CN=freepsw/O=mytem/"

# 3) CSR을 이용하여 인증서(.crt) 생성
## 내가 생성할 인증서를 인증해줄 root CA(인증서)를 지정해야 함. (Minikube /var/lib/localkube/ca.crt)
## Minikube에서는 root CA를 자체적으로 생성하여 관리 (k8s cluster 내부에서 사용하는 인증서의 root CA로 활용)

> sudo openssl x509 -req -in mini-freepsw-csr.pem -CA /var/lib/minikube/certs/ca.crt -CAkey /var/lib/minikube/certs/ca.key -CAcreateserial -out mini-freepsw.crt -day
Signature ok
subject=/CN=freepsw/O=mytem
Getting CA Private Key

> ls
mini-freepsw-csr.pem  mini-freepsw.crt	mini-freepsw.pem

# minikube에서 생성한 인증서를 내 pc/notebook으로 전송한다.
> scp mini-freepsw.crt skiper@192.168.99.1:/Users/skiper/.minikube
$ scp mini-freepsw.pem skiper@192.168.99.1:/Users/skiper/.minikube
```

###### 2-2) 나의 pc에서 minikube에서 활용할 인증서 지정하기
```shell
# minikube에서 전송한 파일 확인
> ls /Users/skiper/.minikube mini-*
mini-freepsw.crt mini-freepsw.pem

# kube config에서 사용자 인증서 경로 변경 (복사한 파일 사용)
# 이제 kubectl로 생성한 사용자는 아래의 인증서(crt, pem)를 통해서 k8s 클러스터의 인증을 하게 될 것이다.
> vi ~/.kube/config
- name: minikube
  user:
    client-certificate: /Users/skiper/.minikube/mini-freepsw.crt
    client-key: /Users/skiper/.minikube/mini-freepsw.pem


> kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "freepsw" cannot list resource "nodes" in API group "" at the cluster scope

```


### 5-5. RBAC
- Authentication이후에 사용자에게 권한(작업, 접근 등)을 지정해야 함. k8s는 기본적으로 이런 권한을 설정하도록 강제함.
- 이러한 제어는 API(kube-apiserver)를 통해 실행 가능
- 사용자가 API 요청(e.g kubectl get nodes 와 같은)을 할때,
- apiserver는 요청한 사용자가 해당 명령을 실행할 권한이 있는지 확인한다.
- 권한 단위
  - Node : kubelets 요청에 대한 권한을 체크
  - ABAC : 속성 기반으로 권한 관리 (e.g alice 사용자는 'marketing' namespace의 모든 권한 사용 가능)
    - 단점으로 매우 세분화된 권한 관리 어려움.
  - RBAC : role기반으로 권한제어, 동적으로 다양한 권한을 부여 가능
  - Webhook : 권한 요청을 외부로 전달 (REST API를 통해서)
    - 자체 권한서버를 관리하는 경우 활용. (권한서버에 대한 입/출력 부하를 별도 관리해야 함)

#### RBAC 적용  
- apiserver 실행시 --authorization-mode를 지정해야 함. (RBAC인 경우 --authorization-mode=RBAC)
- Default로 RBAC가 지정됨.
- minikube start --extra-config=apiserver.Authorization.Mode=RBAC

#### RBAC 추가
- kubectl을 이용하여 RBAC 룰을 추가하여 권한 관리 가능
  - 1) role을 생성하고, (이때 role의 권한, 즉 접근 가능한 namespce 지정 등을 할 수 있다.)
  - 2) role에 사용자와 그룹을 할당한다.
- Role/RoleBinding : 단일 namespace에만 적용
- ClusterRole/ClusterRoleBinding : 클러스터 전체 namespace에 적용
- 아래 예시는 Role을 생성하고, 리소스 및 실행가능 명령어를 제한
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "secrets"]
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: freepsw
  apiGroup: rbac.authorization.k8s.io
```


#### Code Examples

###### 1) Minikube에서 사용자 인증서 생성하기.
```shell
# k8s server로 접속 (cloud vm을 사용하는 경우 ssh로 접속)
> ssh -i ~/.ssh/gcp-freepsw freepsw@34.67.2xx.xxx
> minikube ssh (아래 실습은  minikube 환경에서 실행)

# 1) 새로운 인증 키 생성  
> openssl genrsa -out mini-freepsw.pem 2048

# 2) 인증서 요청(CSR-Certificate Signing Request) 생성
## CSR 은 인증기관에 인증서 발급 요청을 하는 특별한 ASN.1 형식의 파일이며(PKCS#10 - RFC2986)  
## 그 안에는 내 공개키 정보와 사용하는 알고리즘 정보등이 들어 있다. 개인키는 외부에 유출되면 안 되므로 이렇게 특별한 형식의 파일을 만들어서 인증기관에 전달하여 인증서를 발급 받는다.
> openssl req -new -key mini-freepsw.pem -out mini-freepsw-csr.pem -subj "/CN=freepsw/O=mytem/"

# 3) CSR을 이용하여 인증서(.crt) 생성
## 내가 생성할 인증서를 인증해줄 root CA(인증서)를 지정해야 함. (Minikube /var/lib/localkube/ca.crt)
## Minikube에서는 root CA를 자체적으로 생성하여 관리 (k8s cluster 내부에서 사용하는 인증서의 root CA로 활용)

> sudo openssl x509 -req -in mini-freepsw-csr.pem -CA /var/lib/minikube/certs/ca.crt -CAkey /var/lib/minikube/certs/ca.key -CAcreateserial -out mini-freepsw.crt -day
Signature ok
subject=/CN=freepsw/O=mytem
Getting CA Private Key

> ls
mini-freepsw-csr.pem  mini-freepsw.crt	mini-freepsw.pem

# minikube에서 생성한 인증서를 내 pc/notebook으로 전송한다.
> scp mini-freepsw.crt skiper@192.168.99.1:/Users/skiper/.minikube
$ scp mini-freepsw.pem skiper@192.168.99.1:/Users/skiper/.minikube
```

##### 2) 내가 생성한 사용자(freepsw)의 인증서를 활용하는 kubectl 환경을 생성한다.
- minikube 클러스터에 접속하는 용도로 생성 (이후 클러스터 명칭을 변경하여 적용 가능)
```shell
> kubectl config set-credentials freepsw --client-certificate=mini-freepsw.crt --client-key=mini-freepsw.pem
User "freepsw" set.
> kubectl config set-context freepsw --cluster=minikube --user freepsw
Context "freepsw" created.

# 생성된 kubectl config를 확인하면, 아래와 같이 추가됨
> kubectl config view
contexts:
- context:
    cluster: minikube
    user: freepsw
  name: freepsw
......
users:
- name: freepsw
  user:
    client-certificate: /Users/skiper/.minikube/mini-freepsw.crt
    client-key: /Users/skiper/.minikube/mini-freepsw.pem

> kubectl config get-contexts
CURRENT   NAME         CLUSTER                                            AUTHINFO                                           NAMESPACE
          freepsw      minikube                                           freepsw
*         minikube     minikube                                           minikube

> kubectl config use-context freepsw
Switched to context "freepsw".

# 새로운 인증서를 가진 freepsw context로 접속했는데도 아래와 같은 에러가 발생
# freepsw 사용자가 k8s 노드에 접근할 수 있는 권한이 없음.
> kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "freepsw" cannot list resource "nodes" in API group "" at the cluster scope
```

##### 3) 권한이 있는 context로 전환하여, freepsw 사용자에게 권한 부여
```shell
> kubectl config use-context minikube
Switched to context "minikube".

> kubectl create -f users/admin-user.yaml
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```
- admin-user.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole   # cluster role(기본 생성된 role)을 freepsw 사용자에게 부여한다.
  name: cluster-admin
subjects:
- kind: User
  name: "freepsw"
  apiGroup: rbac.authorization.k8s.io
```

##### 4) 권한이 적용되고, node를 조회할 수 있는지 확인
```shell
> kubectl config use-context freepsw
Switched to context "freepsw".

>  kubectl get nodes
NAME       STATUS   ROLES    AGE    VERSION
minikube   Ready    master   100d   v1.17.0
```


##### 5) cluster admin role이 아닌, pod에 대한 조회/업데이트만 권한을 부여하려면? (지정된 namespace에서)
```shell
> kubectl config use-context minikube
# 기존 role은 삭제
> kubectl delete -f users/admin-user.yaml
clusterrolebinding.rbac.authorization.k8s.io "admin-user" deleted

# pod만 접근 가능한 role을 부여함.
> kubectl create -f users/user.yaml
role.rbac.authorization.k8s.io/pod-reader created
rolebinding.rbac.authorization.k8s.io/read-pods created

> kubectl config use-context freepsw
Switched to context "freepsw".

# 이전과 달리 node에 대한 접근권한이 없음.
> kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "freepsw" cannot list resource "nodes" in API group "" at the cluster scope

# pod는 조회 가능함. (현재 생성된 pod가 없음)
> kubectl get pods
No resources found in default namespace.
```

- user.yaml
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"] # pod에 대한 접근 권한만 부여함.
  verbs: ["get", "watch", "list", "create", "update", "patch", "delete"]
- apiGroups: ["extensions", "apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: freepsw
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```


### 5-5. Networking
#### Network 유형
- Container to container : localhost , port number
- Pod to Service : NodePort, DNS
- External to Service : Lodbalancer, NodePort
#### Pod to Pod 통신
- 기본으로 제공 (다른 노드에서도 Pod의s ip 접근 가능 -> 이는 네트워크 구성에 따라 다른 방식을 지원 )
#### On AWS : kubenet networking (kops default)
- 모든 pod의 ip는 VPC를 통해 접근 가능
- k8s master는 각 노드에 /24 subnet을 할당 한다. (256개 ip 주소)
  - /24 = 서브넷 마스크:255.255.255.0
  - = 1111 1111.1111 1111.1111 1111.0000 0000
  - 즉, 24개의 1의 영역은 네트워크 부분이고,
  - 나머지 8개 영역이 IP(호스트) 영역이다.  2^8승은 256
- AWS에는 50개 노드로 제약이 있음. (100개까지 증가할 수 있으나, 성능 이슈 발생)
#### Network 지원 유형
- kubernetes의 네트워크 구성을 지원하는 대안들
- Container Network Interface (CNI)
  - container 내부에서 network interface를 위한 라이브라러/plugin을 제공하는 소프트웨어
  - Calico, Weave
- Overlay Network
  - Flannel (쉽고 대중적으로 사용)
  ![flannel](https://postfiles.pstatic.net/MjAyMDA0MTRfMTcy/MDAxNTg2ODYxNzgyMzcy.2d2BwemXN01XMRathMy3kYRG7F3CzLuR9jtcOe2Qaxcg.YJYcCHU85AtncYIXtnowS8U-9w5oUnMMs0ZOlqkjovgg.PNG.freepsw/image.png?type=w580)

### 5-6. Node Maintenance
- Node Controller가 node 객체에 대한 관리 주체
  - 신규 노드 생성시 ip spce를 할당
  - 가용상 노드 목록 관리
  - 노드 상태 점검 (unhealthy node는 제외하고, 해당 포드는 정상노드로 재배포)
```shell
> kubectl drain mynode --force
```

### 5-7. High Avalability
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/ 참고
- 아래와 같은 설정이 필요
  - clustering etcd : 최소 3개의 etcd node
  - API Server 복제(이중화) : loadbalancer로 연결
  - Scheduler와 Controller 인스턴스를 여러 개 실행

#### Kops 명령어에서 ha 구성 방식
- 아래 명령어에서 zone을 3개 영역으로 입력 함.
```shell
> kops create cluster --name=k8s.freepsw.com \
  --state=s3://kops-state-freepsw --zones=eu-west-1a,eu-west-1b,eu-west-1c \
  --node-count=2 --node-size=t2.micro --master-size=t2.micro \
  --dns-zone=k8s.freepsw.com \
  --master-zones=eu-west-1a,eu-west-1b,eu-west-1c
```

### 5-8. Admission Controller
> https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
- admission controller는 kubernetes API server로 가는 요청을 intercept할 수 있다.
- Interception은 인증(token 또는 인증서 사용)/권한관리(rbac)가 적용된 후에 발생하며,
- API로 요청한 object가 etcd에 생성되기 전에 intercep 된다.
- 이 기능은 cluster 생성 시점에 kubeapserver로 전달되는 aggument로 설정 가능
```
> kube-apiserver --enable-admission-plugins=NamespaceLifecycle,....
```

### 5-9. Pod Security Policies
- pod 생성 및 업데이트 시점에 보안 측면에서 pod를 관리 할 수 있다.
  - pod가 privileged mode로 실행되는 것을 금지
  - mount 가능한 volume 유형을 지정
  - container가 root 권한으로 실행되지 않도록 지정. (가능한 uid/gid 범위를 지정)
- pod security는 admission controller의 하나의 유형이다.

#### Code Examples
- pod-security-policies 아래 README.md



### 5-10. Etcd
- kubernetes의 중요한 데이터를 저장하는 공간 (분산 및 key-value 저장소 )
- gRPC API 지원 , 자동으로 TSL 지원
- 10,000 / sec 의 쓰기 성능
- Raft consensus algorithm 지원 (신뢰성) : https://raft.github.io/, https://swalloow.github.io/raft-consensus
