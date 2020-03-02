# kubernetes_exercise
kubernetes 관련 작업을 정리 (사례 중심)
-https://gist.github.com/edsiper/fac9a816898e16fc0036f5508320e8b4 참고

## Google kubernetes 인증
  * gcloud container clusters get-credentials kuar-cluster --zone asia-northeast1-c --project myproject-209707

## Kubernetes dashboard 설정
- Google에서 제공하는 Dashboard는 외부 네트워크에서 접속이 어려움
- 따라서 kubectl proxy를 이용하여 로컬 머신에서 8001 port로 접속할 수 있도록 설정

### kubernetes dashboard 설치
```
# 사용자 권한 부여
> kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user $(gcloud config get-value account)

> kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

### 로컬 머신이 UI가 없는 서버머신인 경우?
- 자체 브라우저가 없기 때문에, curl 과 같은 명령도구로만 조회가능
- 따라서 내 노트북에서 해당 서버(kubectl proxy를 실행한 서버)에 접속할 수 있도록
- port forwarding을 추가
  - 8002 : 내 노트북에서 접속할 port
  - localhost : kubectl proxy를 통해 설정된 주소
    - 즉, kubectl proxy를 실행하면, localhost:8001이 kubernetes master로 연결된다.
  - 8001 : kubectl proxy에서 제공하는 ports
  - 35.200.87.xxx : kubectl proxy를 실행한 서버
  - text.XX : 35.200.87.xxx에 접속할 계정
```
> ssh -L 8002:localhost:8001 test.XX@35.200.87.xxx
```
- 이제 내 노트북에서 kubernetes dashboard에 접속해 보자
  - http://localhost:8002/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default
- 초기 화면에서 사용자 인증이 나온다. (token 값을 찾아서 입력하자)
```
> kubectl config view
users:
- name: gke_myproject-209707_asia-northeast1-c_kuar-cluster
  user:
    auth-provider:
      config:
        access-token: ya29.Glz5BcvEvUto0X4nfrkm_4_ofZlT2Ns9bQpirsdfcuRNWBjuVi1l83YlQju56YkCaUMrGj7Z-y8P9eKYS3ljBbKpLbMbBdHo_QirZ94-zf6uzQ2sh4ugF3KUpvNIHQ
```


## 3. Check kubernetes status

- 버전확인
```
> kubectl version
Client Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.7", GitCommit:"dd5e1a2978fd0b97d9b78e1564398aeea7e7fe92", GitTreeState:"clean", BuildDate:"2018-04-19T00:05:56Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"10+", GitVersion:"v1.10.4-gke.2", GitCommit:"eb2e43842aaa21d6f0bb65d6adf5a84bbdc62eaf", GitTreeState:"clean", BuildDate:"2018-06-15T21:48:39Z", GoVersion:"go1.9.3b4", Compiler:"gc", Platform:"linux/amd64"}
```

- 클러스터 상태 조회
```
> kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
etcd-1               Healthy   {"health": "true"}   
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}
```

- 워커 노드 목록조회
```
> kubectl get nodes
NAME                                          STATUS    ROLES     AGE       VERSION
gke-kuar-cluster-default-pool-b9eba861-8bx7   Ready     <none>    19m       v1.10.4-gke.2
gke-kuar-cluster-default-pool-b9eba861-k1nw   Ready     <none>    19m       v1.10.4-gke.2
gke-kuar-cluster-default-pool-b9eba861-xmlq   Ready     <none>    19m       v1.10.4-gke.2
```


- 워커 노드 상세정보 조회
```
> kubectl describe nodes gke-kuar-cluster-default-pool-b9eba861-8bx7
Name:               gke-kuar-cluster-default-pool-b9eba861-8bx7
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/fluentd-ds-ready=true
                    beta.kubernetes.io/instance-type=n1-standard-2
                    beta.kubernetes.io/os=linux
                    cloud.google.com/gke-nodepool=default-pool
                    failure-domain.beta.kubernetes.io/region=asia-northeast1
                    failure-domain.beta.kubernetes.io/zone=asia-northeast1-c
                    kubernetes.io/hostname=gke-kuar-cluster-default-pool-b9eba861-8bx7
Annotations:        node.alpha.kubernetes.io/ttl=0
                    volumes.kubernetes.io/controller-managed-attach-detach=true
Taints:             <none>
CreationTimestamp:  Mon, 06 Aug 2018 06:06:08 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  KernelDeadlock       False   Mon, 06 Aug 2018 06:32:24 +0000   Mon, 06 Aug 2018 06:05:15 +0000   KernelHasNoDeadlock          kernel has no deadlock
  NetworkUnavailable   False   Mon, 06 Aug 2018 06:06:21 +0000   Mon, 06 Aug 2018 06:06:21 +0000   RouteCreated                 RouteController created a route
  OutOfDisk            False   Mon, 06 Aug 2018 06:32:21 +0000   Mon, 06 Aug 2018 06:06:08 +0000   KubeletHasSufficientDisk     kubelet has sufficient disk space available
  MemoryPressure       False   Mon, 06 Aug 2018 06:32:21 +0000   Mon, 06 Aug 2018 06:06:08 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Mon, 06 Aug 2018 06:32:21 +0000   Mon, 06 Aug 2018 06:06:08 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Mon, 06 Aug 2018 06:32:21 +0000   Mon, 06 Aug 2018 06:06:08 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Mon, 06 Aug 2018 06:32:21 +0000   Mon, 06 Aug 2018 06:06:28 +0000   KubeletReady                 kubelet is posting ready status. AppArmor enabled
Addresses:
  InternalIP:  10.146.0.5
  ExternalIP:  35.200.105.164
  Hostname:    gke-kuar-cluster-default-pool-b9eba861-8bx7
Capacity:
 cpu:                2
 ephemeral-storage:  98868448Ki
 hugepages-2Mi:      0
 memory:             7658196Ki
 pods:               110
Allocatable:
 cpu:                1930m
 ephemeral-storage:  47093746742
 hugepages-2Mi:      0
 memory:             5778132Ki
 pods:               110
System Info:
 Machine ID:                 9818d5b49b496640f9e5d89f95e88de9
 System UUID:                9818D5B4-9B49-6640-F9E5-D89F95E88DE9
 Boot ID:                    e76b11c1-a44a-4bb5-89db-f5ea5d57ee78
 Kernel Version:             4.14.22+
 OS Image:                   Container-Optimized OS from Google
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://17.3.2
 Kubelet Version:            v1.10.4-gke.2
 Kube-Proxy Version:         v1.10.4-gke.2
PodCIDR:                     10.4.1.0/24
ExternalID:                  8372327194269735918
Non-terminated Pods:         (5 in total)
  Namespace                  Name                                                      CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------                  ----                                                      ------------  ----------  ---------------  -------------
  kube-system                fluentd-gcp-scaler-7c5db745fc-ckjg9                       0 (0%)        0 (0%)      0 (0%)           0 (0%)
  kube-system                fluentd-gcp-v3.0.0-ccdlz                                  100m (5%)     0 (0%)      200Mi (3%)       300Mi (5%)
  kube-system                kube-dns-788979dc8f-n5qf8                                 260m (13%)    0 (0%)      110Mi (1%)       170Mi (3%)
  kube-system                kube-proxy-gke-kuar-cluster-default-pool-b9eba861-8bx7    100m (5%)     0 (0%)      0 (0%)           0 (0%)
  kube-system                metrics-server-v0.2.1-7486f5bd67-lnnrf                    53m (2%)      148m (7%)   154Mi (2%)       404Mi (7%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ------------  ----------  ---------------  -------------
  513m (26%)    148m (7%)   464Mi (8%)       874Mi (15%)
Events:
  Type    Reason                   Age   From                                                     Message
  ----    ------                   ----  ----                                                     -------
  Normal  Starting                 26m   kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7     Starting kubelet.
  Normal  NodeHasSufficientDisk    26m   kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7     Node gke-kuar-cluster-default-pool-b9eba861-8bx7 status is now: NodeHasSufficientDisk
  Normal  NodeHasSufficientMemory  26m   kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7     Node gke-kuar-cluster-default-pool-b9eba861-8bx7 status is now: NodeHasSufficientMemory
  Normal  NodeHasNoDiskPressure    26m   kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7     Node gke-kuar-cluster-default-pool-b9eba861-8bx7 status is now: NodeHasNoDiskPressure
  Normal  NodeHasSufficientPID     26m   kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7     Node gke-kuar-cluster-default-pool-b9eba861-8bx7 status is now: NodeHasSufficientPID
  Normal  NodeAllocatableEnforced  26m   kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7     Updated Node Allocatable limit across pods
  Normal  Starting                 26m   kube-proxy, gke-kuar-cluster-default-pool-b9eba861-8bx7  Starting kube-proxy.
  Normal  NodeReady                25m   kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7     Node gke-kuar-cluster-default-pool-b9eba861-8bx7 status is now: NodeReady
```


## Kubernetes 구성요소 확인

- DNS 서비스
```
> kubectl get deployments --namespace=kube-system kube-dns
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-dns   2         2         2            2           35m
```

- DNS서버에 대한 로드밸런싱을 수행하는 서비스
```
> kubectl get services --namespace=kube-system kube-dns
NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.7.240.10   <none>        53/UDP,53/TCP   38m
```

- Proxy 서비스
```
> kubectl get daemonSets --namespace=kube-system kube-proxy
```

- Kubernetes UI (단일 복제본으로 실행, deployment로 관리)
```
> kubectl get deployments --namespace=kube-system kubernetes-dashboard
- cloud 서비스(google, aws, azure)로 생성한 경우,
- UI서비스가 없다
```


## 4. Kubectl Commands
### Context
- Namespace를 변경관리하는 용도
```
> kubectl config set-context my-context --namespace=mystuff
Context "my-context" created.
> kubectl config use my-context
Switched to context "my-context".
```

### Kubernetes Object 조회
- Rest API를 이용하여 Kubernetes 내부 자원정보 조회
- kubectl get <자원이름> : 모든 자원 조회
- kubectl get <자원이름>  <객체이름> : 특정 자원 조회
- kubectl get pods -o jsonpath --template={.status.podIP} : 특정 path의 정보 추출
- 결과값을 특정 포맷으로 받기
  * kubectl get <자원이름> -o json 또는 yaml
```
> kubectl get services --namespace=kube-system kube-dns -o json
{
    "apiVersion": "v1",
    "kind": "Service",
    "metadata": {
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Service\",\"metadata\":{\"annotations\":{},\"labels\":{\"addonmanager.kubernetes.io/mode\":\"Reconcile\",\"k8s-app\":\"kube-dns\",\"kubernetes.io/cluster-service\":\"true\",\"kubernetes.io/name\":\"KubeDNS\"},\"name\":\"kube-dns\",\"namespace\":\"kube-system\"},\"spec\":{\"clusterIP\":\"10.7.240.10\",\"ports\":[{\"name\":\"dns\",\"port\":53,\"protocol\":\"UDP\"},{\"name\":\"dns-tcp\",\"port\":53,\"protocol\":\"TCP\"}],\"selector\":{\"k8s-app\":\"kube-dns\"}}}\n"
        },
        "creationTimestamp": "2018-08-06T06:05:57Z",
        "labels": {
            "addonmanager.kubernetes.io/mode": "Reconcile",
            "k8s-app": "kube-dns",
            "kubernetes.io/cluster-service": "true",
            "kubernetes.io/name": "KubeDNS"
        },
        "name": "kube-dns",
        "namespace": "kube-system",
        "resourceVersion": "280",
        "selfLink": "/api/v1/namespaces/kube-system/services/kube-dns",
        "uid": "c8cc617c-993e-11e8-b2b5-42010a920098"
    },
    "spec": {
        "clusterIP": "10.7.240.10",
        "ports": [
            {
                "name": "dns",
                "port": 53,
                "protocol": "UDP",
                "targetPort": 53
            },
            {
                "name": "dns-tcp",
                "port": 53,
                "protocol": "TCP",
                "targetPort": 53
            }
        ],
        "selector": {
            "k8s-app": "kube-dns"
        },
        "sessionAffinity": "None",
        "type": "ClusterIP"
    },
    "status": {
        "loadBalancer": {}
    }
}s
```

- kybectl 상태정보 조회
```
> kubectl describe <자원이름> <객체이름>
```

### kubernetes 객체(yaml, json) 생성 및 업데이트, 삭제
```
- obj.yaml에 정의된 객체 생성
> kubectl apply -f obj.yaml

- obj.yaml에 정의된 객체로 업데이트
> kubectl apply -f obj.yaml

- 생성된 객체 삭제 (30초 동안은 종료 유예기간을 가지고, Terminating으로 표시됨. )
> kubectl delete <자원이름> <객체이름>

### Label 과 annotations
```
- label 업데이트 (bar pod의 label에 color=red 추가)
> kubectl label pods bar color=red

- label 삭제 (color 라벨 삭제)
> kubectl label pods bar -color
```

### Debugging
```
- 동작중인 컨테이너 로그 확인 (-c 옵션으로 특정 컨테이너 로그 조회 가능)
> kubectl logs <pod명>

- 동작중인 컨테이너 접속
> kubectl exec -it <pod name> ash

- 컨테이너로 파일 복사
> kubectl cp <pod name>:path/to/remote/file /path/to/local/file

## 5. POD Commands

```
>kubectl run kuard --image=gcr.io/kuar-demo/kuard-amd64:1
deployment "kuard" created

> kubectl get pods
NAME                    READY     STATUS    RESTARTS   AGE
kuard-b75468d67-z79v2   1/1       Running   0          11s

> kubectl delete deployments/kuard
deployment "kuard" deleted
```

### pod instance 생성
- manifest 파일에 pod 정의 (5-1-kuard-pod.yaml )
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
spec:
  containers:
  - image: gcr.io/kuar-demo/kuard-amd64:1
    name: kuard
    ports:
    - containerPort: 8080
      name: http
      protocol: TCP
```
- pod 인스턴스 실헹 / 조회/ 삭제
```
> kubectl apply -f 5-1-kuard-pod.yaml
pod "kuard" created

> kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
kuard     1/1       Running   0          6s

> kubectl describe pod kuard
Name:         kuard
Namespace:    default
Node:         gke-kuar-cluster-default-pool-b9eba861-8bx7/10.146.0.5
Start Time:   Mon, 06 Aug 2018 07:36:26 +0000
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"kuard","namespace":"default"},"spec":{"containers":[{"image":"gcr.io/kuar-demo/kua...
              kubernetes.io/limit-ranger=LimitRanger plugin set: cpu request for container kuard
Status:       Running
IP:           10.4.1.7
Containers:
  kuard:
    Container ID:   docker://71eb3e14b8cc315cec631d3e03056e6e4599c20a5f123a0c0fc0b486bd807238
    Image:          gcr.io/kuar-demo/kuard-amd64:1
    Image ID:       docker-pullable://gcr.io/kuar-demo/kuard-amd64@sha256:3e75660dfe00ba63d0e6b5db2985a7ed9c07c3e115faba291f899b05db0acd91
    Port:           8080/TCP
    State:          Running
      Started:      Mon, 06 Aug 2018 07:36:29 +0000
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        100m
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-9q7k4 (ro)
Conditions:
  Type           Status
  Initialized    True
  Ready          True
  PodScheduled   True
Volumes:
  default-token-9q7k4:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-9q7k4
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason                 Age   From                                                  Message
  ----    ------                 ----  ----                                                  -------
  Normal  Scheduled              2m    default-scheduler                                     Successfully assigned kuard to gke-kuar-cluster-default-pool-b9eba861-8bx7
  Normal  SuccessfulMountVolume  2m    kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7  MountVolume.SetUp succeeded for volume "default-token-9q7k4"
  Normal  Pulling                2m    kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7  pulling image "gcr.io/kuar-demo/kuard-amd64:1"
  Normal  Pulled                 2m    kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7  Successfully pulled image "gcr.io/kuar-demo/kuard-amd64:1"
  Normal  Created                2m    kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7  Created container
  Normal  Started                2m    kubelet, gke-kuar-cluster-default-pool-b9eba861-8bx7  Started container

> kubectl delete pods/kuard
pod "kuard" deleted
```

- port-forward를 이용한 원격접속
  * localhost에서 8080 port를 통해, kubernetes의 pod에 접속할 수 있다.
```
> kubectl port-forward kuard 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
```

- 관련 명령어

```
- 로그 확인
> kubectl logs kuard

- 컨테이너 내부 접속
> kubectl exec -it kuard ash

- kubectl cp <pod name>:/remote-flename.txt ./local-filename.txt
```


### 상태관리 (Heartbeat)
- Service 상태 체크 Heartbeat  
   * https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
   * Liveness probe: 컨테이너가 서비스가 가능한 상태인지를 체크 (deadlock 상태, 서비스 완전 정지상태, 재시작 외에는 정상화 어려움)
    > 상태조건에 맞지 않으면, 컨테이너를 재시작 하려는 경우 사용. (서비스 불능 상태)
   * Readiness probe: 컨테이너가 살아 있는지 아닌지를 체크 (일시적 서비스 부하, 초기 데이터로딩 및 대용량 처리 등, )
    > 일시적인 응답 지연으로, 컨테이너를 재시작 하지는 않고, 요청을 다른 컨테이너로 분산 (요청에서 제외)


## 6. Label & Annotations
### Label
- 복잡하고, 다양한 pod(container)를 효율적인 집합으로 다루기 위한 방법
- Key : value 로 구성
- 접두사 / 이름 (영문자로 시작, -,_ 사용가능, 63 이하 문자열 )
  * acme.com/app-version=1.0.0

```
# Label 생성 (deployment controller를 생성하면서 label 생성 )
> kubectl run alpha-prod --image=gcr.io/kuar-demo/kuard-amd64:1 --replicas=2 --labels="ver=1,app=alpaca,env=prod"
> kubectl run alpha-test --image=gcr.io/kuar-demo/kuard-amd64:1 --replicas=2 --labels="ver=2,app=alpaca,env=test"

# Label 조회
> kubectl get deployments --show-labels
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       LABELS
alpha-prod   2         2         2            2           48s       app=alpaca,env=prod,ver=1

# Label 추가
> kubectl label deployments alpha-prod "canary=true"
deployment "alpha-prod" labeled

# Label 삭제
> kubectl label deployments alpha-prod "canary-"
deployment "alpha-prod" labeled

# Label Selector를 이용하여 deployment 조회
# key=value, key!=value, key in (value1, value2), key notin (value1, value2), key, !key(키가 설정되지 않은 경우)
> kubectl get deployment --selector="app=alpaca"
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
alpha-prod   2         2         2            2           21m
alpha-test   2         2         2            2           57s
```

### Annotations
- Label과 달리 추가적인 메타정보를 저장하는 목적
  - Label : 객체를 식별하고, 그룹화
  - Annotations : API를 통해 추가적인 데이터를 저장하는 방법 (설정정보 전달 및 도구에 대한 정보 제공), Label과 일부 기능은 겹침
    - 객체에 대한 업데이트 사유 추적
    - 특정 스케줄링 정책 전달
    - 예시 (특정 아이콘의 url 주소를 제공)
    ```
    metadata:
      annotations:
        examples.com/icon-url: "https://example.com/icon.png"
    ```

## 7. Service Discovery
- 수평적 포드 확장, RC Set 자동 확장 등의 문제(부하에 따라 포드 수 조정 등)를 해결
- 프로세스 목록에서 서비스에 할당된 것을 찾아 문제를 해결하도록 돕는다.
### Service Object (ClusterIP)

```
> kubectl run alpaca-prod --image=gcr.io/kuar-demo/kuard-amd64:1 --replicas=3 --port=8080 --labels="ver=1,app=alpaca,env=prod"
# deployment(alpaca-prod)를 서비스로 등록
> kubectl expose deployment alpaca-prod
> kubectl run bandicoot-prod --image=gcr.io/kuar-demo/kuard-amd64:1 --replicas=2 --port=8080 --labels="ver=2,app=bandicoot,env=prod"
> kubectl expose deployment bandicoot-prod
> kubectl get services -o wide
# 아래 cluster_ip가 할당됨. 이는 selector로 식별되는 모든 pod로 로드밸런싱하는 특별한 목적의 IP
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE       SELECTOR
alpaca-prod      ClusterIP   10.7.253.165   <none>        8080/TCP   2m        app=alpaca,env=prod,ver=1
bandicoot-prod   ClusterIP   10.7.244.235   <none>        8080/TCP   55s       app=bandicoot,env=prod,ver=2
kubernetes       ClusterIP   10.7.240.1     <none>        443/TCP    6h        <none>

# 생성된 service에 연결할 수 있도록 port forwarding을 한다.
# 먼저 port-forwarding할 pod의 name을 추춣

> kubectl get pods -l app=alpaca -o json
{
    "apiVersion": "v1",
    "items": [
        {
            "apiVersion": "v1",
            "kind": "Pod",
            "metadata": {
                "annotations": {
                    "kubernetes.io/limit-ranger": "LimitRanger plugin set: cpu request for container alpaca-prod"
                },
                "creationTimestamp": "2018-08-07T06:31:25Z",
                "generateName": "alpaca-prod-7f94b54866-",
                "labels": {
                    "app": "alpaca",
                    "env": "prod",
                    "pod-template-hash": "3950610422",
                    "ver": "1"
                },
                "name": "alpaca-prod-7f94b54866-6lkk9",  --> 이걸 찾아야 함.
                "namespace": "default",
                "ownerReferences": [
                    {
                        "apiVersion": "extensions/v1beta1",
                        "blockOwnerDeletion": true,
                        "controller": true,
                        "kind": "ReplicaSet",
                        "name": "alpaca-prod-7f94b54866",
                        "uid": "81e97159-9a0b-11e8-b0da-42010a9200b6"
                    }
                ],
                "resourceVersion": "32660",
                "selfLink": "/api/v1/namespaces/default/pods/alpaca-prod-7f94b54866-6lkk9",
                "uid": "81ee963a-9a0b-11e8-b0da-42010a9200b6"
            },
            ....

# 많은 json 값중에서 특정 값만 찾을 수 있도록 jsonpath를 사용
> kubectl get pods -l app=alpaca -o jsonpath='{.items[0].metadata.name}'
alpaca-prod-7f94b54866-6lkk9

# 이를 내부 변수에 저장하여 port-forwarding에 활용하자
> ALPACA_POD=$(kubectl get pods -l app=alpaca -o jsonpath='{.items[0].metadata.name}')
> kubectl port-forward $ALPACA_POD 48858:8080
Forwarding from 127.0.0.1:48858 -> 8080

# endpoint는 서비스에서 트래픽을 전송하는 대상을 찾기 위한 하위수준의 방법
> kubectl get endpoints alpaca-prod --watch
NAME          ENDPOINTS                                   AGE
alpaca-prod   10.4.0.5:8080,10.4.0.6:8080,10.4.2.6:8080   10m
```

### Cluster 외부로 서비스 (NodePort)
- Service Object의 spec.type을 NodePort로 지정하여
- 모든 노드에서 특정 포트를 통해 pod에 접속할 수 있다.
- NodePort는 하드웨어/소프트웨어 및 로드밸런서와 연계하여, 더 좋은 서비스를 제공 가능



```
# service object에 대한 spec을 바로 수정 및 적용하는 명령어.
> kubectl edit service alpaca-prod
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: 2018-08-07T23:42:00Z
  labels:
    app: alpaca
    env: prod
    ver: "1"
  name: alpaca-prod
  namespace: default
  resourceVersion: "1542"
  selfLink: /api/v1/namespaces/default/services/alpaca-prod
  uid: 7ae04972-9a9b-11e8-a042-42010a920014
spec:
  clusterIP: 10.7.250.175
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: alpaca
    env: prod
    ver: "1"
  sessionAffinity: None
  type: NodePort # 여기를 NodePort로 변경.
status:
  loadBalancer: {}
```
- spec.type을 NodePort로 변경한 후 정보
```
# 변경 후 정보 (NodePort 항목이 추가되고, port가 지정됨)
kubectl describe service alpaca-prod
Name:                     alpaca-prod
Namespace:                default
Labels:                   app=alpaca
                          env=prod
                          ver=1
Annotations:              <none>
Selector:                 app=alpaca,env=prod,ver=1
Type:                     NodePort
IP:                       10.7.250.175
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30857/TCP  # 추가된 포트
Endpoints:                10.4.0.5:8080,10.4.0.6:8080,10.4.2.6:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>


# spec.type이 ClusterIP인 경우 정보
kubectl describe service alpaca-prod
Name:              alpaca-prod
Namespace:         default
Labels:            app=alpaca
                   env=prod
                   ver=1
Annotations:       <none>
Selector:          app=alpaca,env=prod,ver=1
Type:              ClusterIP
IP:                10.7.250.175
Port:              <unset>  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.4.0.5:8080,10.4.0.6:8080,10.4.2.6:8080
Session Affinity:  None
Events:            <none>
```
- 이렇게 노드별로 port를 할당하여, 해당 서비스가 pod에 연결될 수 있음
- ssh port-forwarding을 이용하여, node에서 8080port로 접속할 수 있도록 설정
- 이 방식으로 node에서 8080으로 접속하면,
- 각 요청은 서비스를 제공하는 pod중 1개로 무작위로 연결된다.
```
> ssh <node ip>  -L 8080:localhost:30875
```

### 클라우드 서비스 활용 (LoadBalancer)
- 클라우드 사업자에 의해 운영되는 서버인 경우, 로드밸런서를 사용
- 별도의 로드밸런서를 구성하여, Nodeport르 생성하고, 이를 클러스터 노드에 전달한다.
- 아레와 같이 EXTERNAL-IP가 할당되고,
- 외부에서 http://35.200.102.69:8080으로 접속하면,
- Pod중 1개로 연결되는 것을 확인할 수 있다.

```
> kubectl get services
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
alpaca-prod   LoadBalancer   10.7.250.175   35.200.102.69   8080:30857/TCP   1h
```

### 기타 상세 정보
#### endpoint
- 모든 service object에 대하여, kubernetes는 endpoint 객체를 생성한다.
```
>  kubectl describe endpoints alpaca-prod
Name:         alpaca-prod
Namespace:    default
Labels:       app=alpaca
              env=prod
              ver=1
Annotations:  <none>
Subsets:
  Addresses:          10.4.0.5,10.4.0.6,10.4.2.6
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    <unset>  8080  TCP
```
- 이 endpoint는 실제 서비스를 하는 deployment(pods)가 삭제되면
- endpoint가 사라지거나, 변경될 수 있다.
- 이를 지속적으로 관찰할 수 있도록, kubernetes에서는 watch로 알람을 받도록 한다.
```
# 현재 endpoint가 할당된 상태
> kubectl get endpoints alpaca-prod --watch
NAME          ENDPOINTS                                   AGE
alpaca-prod   10.4.0.5:8080,10.4.0.6:8080,10.4.2.6:8080   1h

# 할당된 deployment를 삭제
>kubectl delete deployment alpaca-prod
deployment "alpaca-prod" deleted

> kubectl get endpoints alpaca-prod --watch
NAME          ENDPOINTS                                   AGE
alpaca-prod   10.4.0.5:8080,10.4.0.6:8080,10.4.2.6:8080   1h
alpaca-prod   10.4.0.5:8080   1h
alpaca-prod   <none>    1h

# 다시 deployment를 생성
> kubectl run alpaca-prod --image=gcr.io/kuar-demo/kuard-amd64:1 --replicas=3 --port=8080 --labels="ver=1,app=alpaca,env=prod"
deployment "alpaca-prod" created

> kubectl get endpoints alpaca-prod --watch
NAME          ENDPOINTS                                   AGE
alpaca-prod   10.4.0.5:8080,10.4.0.6:8080,10.4.2.6:8080   1h
alpaca-prod   10.4.0.5:8080   1h
alpaca-prod   <none>    1h
alpaca-prod   10.4.2.7:8080   1h
alpaca-prod   10.4.0.8:8080,10.4.2.7:8080   1h
alpaca-prod   10.4.0.7:8080,10.4.0.8:8080,10.4.2.7:8080   1h

```

- label을 이용한 객체 삭제 (app가 포함된 모든 service, deployment 삭제)
```
> kubectl delete services,deployments -l app
```


## ReplicaSet
- pod이 복제 집합을 쉽게 생성 및 관리 할 수 있는 pod 관리자 역할.

```
# RS 생성
> kubectl apply -f 8-1-kuard-rs.yaml

> kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
kuard-2n65h   1/1       Running   0          7s

> kubectl describe rs kuard
Name:         kuard
Namespace:    default
Selector:     app=kuard,version=2
Labels:       app=kuard
              version=2
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"extensions/v1beta1","kind":"ReplicaSet","metadata":{"annotations":{},"name":"kuard","namespace":"default"},"spec":{"replicas":1,"templat...
Replicas:     1 current / 1 desired
Pods Status:  1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=kuard
           version=2
  Containers:
   kuard:
    Image:        gcr.io/kuar-demo/kuard-amd64:2
    Port:         <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
```

```
# pod 조회
> kubectl get pods -l app=kuard,version=2
```


### Pod 자동확장
- cpu, memory등의 자원사용량이 증가하면, 자동으로 확장할 수 있도록 설정 가능
- CPU 사용률이 80%인 복제본을 2개 ~ 5개까지 확장할 수 있도록 설정
#### CPU 기반 자동확장
```
> kubectl autoscale rs kuard --min=2 --max=5 --cpu-percent=80
replicaset "kuard" autoscaled

# 아래 명령으로 자원사용 현황 조회
> kubectl get  hpa
NAME      REFERENCE          TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
kuard     ReplicaSet/kuard   0% / 80%   2         5         2          3m

> kubectl delete rs kuard
replicaset "kuard" deleted

# pod는 삭제하지 않고, RS만 지울 경우
> kubectl delete rs kuard --cascade=false
```

## 9. daemonSets
- ReplicaSet처럼 pod 복제로 중복하여 서비스를 확장하는 것이 아닌,
- 노드별로 하나의 작업(Agent, daemon)을 실행하려는 목적으로 활용.
- Node selector를 사용하지 않으면, 기본적으로 모든 노드에 1개씩 pod를 설치함.
### 이슈
- 실제 9-1.fluentd.yaml을 실행해보니, 노드별로 pod가 생성되지 않음. (daemonset만 생성됨.)
- 9-1-fluentd.yaml
```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    app: fluentd
spec:
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v0.14.10
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```
```
> kubectl apply -f 9-1-fluentd.yaml
daemonset "fluentd" created
```

### 해결?
- Node에 Label을 추가하고,
- Daemonset의 NodeSelector에 Node의 label을 지정하여,
- 해당 Node에만 DaemonSet이 설치되도록 수정
- https://www.mirantis.com/blog/scaling-kubernetes-daemonsets/ 참고
- test.yaml
```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: frontend
spec:
  template:
    metadata:
      labels:
        app: frontend-webserver
    spec:
      nodeSelector:
        app: frontend-node
      containers:
        - name: webserver
          image: nginx
          ports:
          - containerPort: 80
```

```
> kubectl apply -f test.yaml
daemonset "frontend" created

# Node에 label을 부여  
> kubectl get nodes
NAME                                          STATUS    ROLES     AGE       VERSION
gke-kuar-cluster-default-pool-96a8e6ca-4bb0   Ready     <none>    11m       v1.10.4-gke.2
gke-kuar-cluster-default-pool-96a8e6ca-8xf4   Ready     <none>    11m       v1.10.4-gke.2
gke-kuar-cluster-default-pool-96a8e6ca-znzt   Ready     <none>    11m       v1.10.4-gke.2

# 1개 노드에 label(app=frontend-node)을 추가 (3개의 노드에 동일한 방식으로 실행 )
> kubectl label node gke-kuar-cluster-default-pool-96a8e6ca-4bb0 app=frontend-node
node "gke-kuar-cluster-default-pool-96a8e6ca-4bb0" labeled

# label이 추가된 노드에 daemonset pod가 실행되는지 확인
> kubectl get nodes
NAME                                          STATUS    ROLES     AGE       VERSION
gke-kuar-cluster-default-pool-96a8e6ca-4bb0   Ready     <none>    11m       v1.10.4-gke.2

# DaemonSet 상세정보 확인
> kubectl describe ds frontend
Name:           frontend
Selector:       app=frontend-webserver
Node-Selector:  app=frontend-node
Labels:         app=frontend-webserver
Annotations:    <none>
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 3
Number of Nodes Misscheduled: 0
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=frontend-webserver
  Containers:
   webserver:
    Image:        nginx
    Port:         80/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                  Message
  ----    ------            ----  ----                  -------
  Normal  SuccessfulCreate  42m   daemonset-controller  Created pod: frontend-dng4m
  Normal  SuccessfulCreate  41m   daemonset-controller  Created pod: frontend-b9rjh
  Normal  SuccessfulCreate  41m   daemonset-controller  Created pod: frontend-c52l6
```

### Rolling Update 방식
- 각 DaemonSet의 어플리케이션을 중단없이 업데이트 하려고 할때 사용함.
- spec.minReadySeconds : 다음 포드를 업데이트 하기전, 포드가 준비(ready) 상태로 있어야 하는 최소시간
- spec.updateStrategy.RollingUpdate.maxUnavailable : 롤링 업데이트시 동시에 업데이트 될 포드의 수 지정
  * 동시에 여러개를 업데이트 하면, 업데이트 속도는 빨라짐. (하지만 업데이트가 실패하면, 전체 서비스에 영향)
```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: frontend
spec:
  updateStrategy: RollingUpdate
    maxUnavailable: 1
    minReadySeconds: 0
  template:
    metadata:
      labels:
        app: frontend-webserver
    spec:
      nodeSelector:
        app: frontend-node
      containers:
        - name: webserver
          image: nginx
          ports:
          - containerPort: 80
```


## 10. Job
- 지금까지는 장기로 실행되는 서비스를 위한 기능을 보았고,
- Job은 일회성으로 실행되는 작업을 처리하기 위한 용도로 설계되었다.
- Job에서는 최종 리턴값(종료코드 = 0)이 성공이 될때 까지 포드를 실행한다.
### Job 패턴
- 기본적으로 job은 성공코드를 리턴할 때 까지 단일 포드를 한번만 실행함.
  - completions : 작업 완료 수 , 몇번의 작업을 실행할 것인지.
  - parallelism : 병렬로 실행할 포드 수
  - "완료까지 1회 실행인 경우는 두 값이 모두 1"

  - One-shot : ( C:1, P:1 ), 데이터베이스 전환 같이 1회만 수행.
  - 특정 환료횟수 까지 병렬 처리 : (C:1+, P:1+), 1개 이상의 포드가 1번 이상 실행
  - 작업 대기열 : (C:1, P:2+), 여러 포드가 중앙작업 대기열을 처리. 성공시까지 하나 이상의 포드가 1번 실행.

#### One-shot
##### 1. kubectl 을 이용하여 oneshot job 생성
```
> kubectl run -i oneshot \
> --image=gcr.io/kuar-demo/kuard-amd64:1 \
> --restart=OnFailure \
> -- --keygen-enable \
>    --keygen-exit-on-complete \
>    --keygen-num-to-gen 10
If you don't see a command prompt, try pressing enter.
2018/08/08 07:14:26 (ID 0 1/10) Item done: SHA256:xCabaPMtxFGma8YsCQhXFiYp/i5FhoCn0w4nB7G1IQw
2018/08/08 07:14:30 (ID 0 2/10) Item done: SHA256:AiI6FGoWMB6l/InYphZwvOqHVriNoNvv0togySdUPtU
2018/08/08 07:14:31 (ID 0 3/10) Item done: SHA256:sE4TggiQZdeqf1H2qlJV1fSr5L1AOhlXfoLIZZpCL5Y
2018/08/08 07:14:33 (ID 0 4/10) Item done: SHA256:33dZdkgyd4aTgS4k0ViufsBg3UzJWz48w2LjREYzTMk
2018/08/08 07:14:37 (ID 0 5/10) Item done: SHA256:Ugs8JLHqv+h/CNwqyXAsXkbvs5hkP1LSWXRsb9aj8k8
2018/08/08 07:14:46 (ID 0 6/10) Item done: SHA256:ISh6ChFZtYLVi1vq77qmb6yaasYOnwOkAL361swEibw
2018/08/08 07:14:50 (ID 0 7/10) Item done: SHA256:oRN4HeC8enEipF/TekA6D4EhzhqnQrkTPhGqXd/Em9s
2018/08/08 07:14:51 (ID 0 8/10) Item done: SHA256:xJoRPbUOjxIN/y3O1uqGZH8BoLYE3QMJSgfIRuS9eoA
2018/08/08 07:14:52 (ID 0 9/10) Item done: SHA256:m5ka3iZs9+xD5LA99F4oZtujUtfrBdVZFtZ6C5u94wY
2018/08/08 07:15:01 (ID 0 10/10) Item done: SHA256:wajyXC7M9vAEVzGpGGQno9msOxBi61UfW012E38rQQQ
2018/08/08 07:15:01 (ID 0) Workload exiting

> kubectl get pod
NAME             READY     STATUS    RESTARTS   AGE
frontend-b9rjh   1/1       Running   0          1h
frontend-c52l6   1/1       Running   0          1h
frontend-dng4m   1/1       Running   0          1h

> kubectl get jobs
NAME      DESIRED   SUCCESSFUL   AGE
oneshot   1         1            1m

```

#####  2. yaml 파일을 이용하여 job 생성
```
> kubectl apply -f 10-1-job-oneshot.yaml
job "oneshot" created

> kubectl describe jobs oneshot
Name:           oneshot
Namespace:      default
Selector:       controller-uid=2ec76fc6-9adb-11e8-978c-42010a920014
Labels:         chapter=jobs
Annotations:    kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"labels":{"chapter":"jobs"},"name":"oneshot","namespace":"default"},"spec":{"templat...
Parallelism:    1
Completions:    1
Start Time:     Wed, 08 Aug 2018 07:18:00 +0000
Pods Statuses:  1 Running / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  chapter=jobs
           controller-uid=2ec76fc6-9adb-11e8-978c-42010a920014
           job-name=oneshot
  Containers:
   kuard:
    Image:  gcr.io/kuar-demo/kuard-amd64:1
    Port:   <none>
    Args:
      --keygen-enable
      --keygen-exit-on-complete
      --keygen-num-to-gen=10
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  17s   job-controller  Created pod: oneshot-c5lwl

> kubectl logs oneshot-c5lwl
2018/08/08 07:18:05 Starting kuard version: v0.7.2-1
2018/08/08 07:18:05 **********************************************************************
2018/08/08 07:18:05 * WARNING: This server may expose sensitive
2018/08/08 07:18:05 * and secret information. Be careful.
2018/08/08 07:18:05 **********************************************************************
2018/08/08 07:18:05 Could not find certificates to serve TLS
2018/08/08 07:18:05 Serving on HTTP on :8080
2018/08/08 07:18:05 (ID 0) Workload starting
2018/08/08 07:18:08 (ID 0 1/10) Item done: SHA256:Xi9mSsNAXq+zFlhioLeYW8JHPAYYY/49ynuo4C1WHgE
2018/08/08 07:18:11 (ID 0 2/10) Item done: SHA256:ssPyUmdR0BMvlfIC1kMZhB0NETpOXnf1mmqSsecV6V4
2018/08/08 07:18:14 (ID 0 3/10) Item done: SHA256:ob8f6dTCKgXbQ8YkFlke6XiDeAIn7Bs/d5c9s5ApsaE
2018/08/08 07:18:21 (ID 0 4/10) Item done: SHA256:x6GHXJSLXZLDeMSKqLqeJHeqVu/7F8NXaSkmPJK+PSw
2018/08/08 07:18:30 (ID 0 5/10) Item done: SHA256:ILTlJmCgitt45ppCK4UGIZDQ6bNCjZeqEb/v/sqeOs0
2018/08/08 07:18:35 (ID 0 6/10) Item done: SHA256:E61qEz6uwaFNwqnuj+n3mn6IDVk7/t0hAYWc6RXSGWw
2018/08/08 07:18:45 (ID 0 7/10) Item done: SHA256:lVnBKQO7PlD26Wq9zmwiWmDhm0Wt/Hj1AhL1yaY39d8
2018/08/08 07:18:47 (ID 0 8/10) Item done: SHA256:18mf/D5/iVoVTn2SuoWnoWVmyJspy9STDYELY/i/1Cc
2018/08/08 07:18:50 (ID 0 9/10) Item done: SHA256:4WJLIpBgS//hHrMywMWl9Jp5OEI7w9cQXGgtCwT7/Jo
2018/08/08 07:18:51 (ID 0 10/10) Item done: SHA256:59ime0hgDfCmqMhsK268EMS+5DAOYNSNrcF6Ksg7X/w
2018/08/08 07:18:51 (ID 0) Workload exiting
```

#####  3. 실패처리
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: oneshot
  labels:
    chapter: jobs
spec:
  template:
    metadata:
      labels:
        chapter: jobs
    spec:
      containers:
      - name: kuard
        image: gcr.io/kuar-demo/kuard-amd64:1
        imagePullPolicy: Always
        args:
        - "--keygen-enable"
        - "--keygen-exit-on-complete"
        - "--keygen-exit-code=1"
        - "--keygen-num-to-gen=3"
      restartPolicy: OnFailure
```
- OnFailure Mode : 해당 pod를 재시작하여 job을 처리함.
```
> ubectl delete jobs oneshot
job "oneshot" deleted

> kubectl apply -f 10-2-job-oneshot-failure1.yaml
job "oneshot" created

# job이 3번 재시작 되었다.
> kubectl get pod -l job-name=oneshot
NAME            READY     STATUS    RESTARTS   AGE
oneshot-m4n9s   0/1       Error     3          1m  
```

- Never Mode: 실패하면 새롭게 Pod를 생성하여 job을 처리
  * 이 방식은 pod가 계속 생성되므로, 불필요한 객체가 많이 생성된다.
  * 따라서 OnFailure를 사용하는 것을 권고.
```
> kubectl get pods -l job-name=oneshot -a
NAME            READY     STATUS    RESTARTS   AGE
oneshot-2vsq2   0/1       Error     0          40s
oneshot-82gf6   0/1       Error     0          30s
oneshot-hhxkb   0/1       Error     0          48s
```

### 병렬 처리
- 키를 동시 생성할 수 있도록 병렬 처리 실행
  * completions : 10 (작업을 10번 실행)
  * parallelism : 5 (동시에 5개의 pod가 실행됨.)
```
# 5개의 pod가 동시에 실행되고 있다.
>  kubectl get pods -o wide
NAME             READY     STATUS    RESTARTS   AGE       IP          NODE
parallel-7dctz   1/1       Running   0          2m        10.4.0.46   gke-kuar-cluster-default-pool-96a8e6ca-4bb0
parallel-b4g64   1/1       Running   0          1m        10.4.1.49   gke-kuar-cluster-default-pool-96a8e6ca-znzt
parallel-klk9d   1/1       Running   0          2m        10.4.0.48   gke-kuar-cluster-default-pool-96a8e6ca-4bb0
parallel-x62hs   1/1       Running   0          1m        10.4.1.48   gke-kuar-cluster-default-pool-96a8e6ca-znzt
parallel-zs4jq   1/1       Running   0          2m        10.4.0.47   gke-kuar-cluster-default-pool-96a8e6ca-4bb0

# pod가 실행되는 이력 확인 (watch 옵션)
> kubectl get pods -w
NAME             READY     STATUS    RESTARTS   AGE
parallel-4msjk   1/1       Running   0          10s
parallel-5xrq4   1/1       Running   0          10s
parallel-7dctz   1/1       Running   0          10s
parallel-klk9d   1/1       Running   0          10s
parallel-zs4jq   1/1       Running   0          10s
oneshot-hdps2   0/1       Error     0         29s
oneshot-4rlxr   0/1       Pending   0         0s
oneshot-4rlxr   0/1       Pending   0         0s
oneshot-4rlxr   0/1       ContainerCreating   0         0s
oneshot-4rlxr   1/1       Running   0         3s
parallel-4msjk   0/1       Completed   0         58s
parallel-x62hs   0/1       Pending   0         0s
parallel-x62hs   0/1       Pending   0         0s
parallel-x62hs   0/1       ContainerCreating   0         0s
oneshot-4rlxr   0/1       Error     0         35s
oneshot-jk6bj   0/1       Pending   0         0s
oneshot-jk6bj   0/1       Pending   0         0s
oneshot-jk6bj   0/1       ContainerCreating   0         0s
parallel-x62hs   1/1       Running   0         2s
oneshot-jk6bj   1/1       Running   0         3s
parallel-5xrq4   0/1       Completed   0         1m
.....

> kubectl delete job parallel
job "parallel" deleted
> kubectl delete job oneshot
job "oneshot" deleted
```

## 11. ConfigMap & Secret
### ConfigMap
- 작은 파일시스템을 정의하는 kubernetes 객체
- 컨테이너의 환경/명령어를 정의하는 변수의 집합
- Pod는 실행 전에 ConfigMap과 결합되어 실행 -> 즉, ConfigMap만 변경하면, 컨테이너 이미지와 포드 재사용이 가능
- 11-1-simple-config.txt
```
parameter1 = value1
parameter2 = value2
```

```
kubectl create configmap my-config \
> --from-file=11-1-simple-config.txt \
ㄱ>--from-literal=extra-param=extra-value \
> --from-literal=another-param=another-value
configmap "my-config" created

> kubectl get configmap my-config -o yaml
apiVersion: v1
data:
  11-1-simple-config.txt: |
    parameter1 = value1
    parameter2 = value2
  another-param: another-value
  extra-param: extra-value
kind: ConfigMap
metadata:
  creationTimestamp: 2018-08-08T08:09:10Z
  name: my-config
  namespace: default
  resourceVersion: "15453"
  selfLink: /api/v1/namespaces/default/configmaps/my-config
  uid: 5450a802-9ae2-11e8-978c-42010a920014
```

#### ConfigMap 사용
- 1. FileSystem : 포드에 configmap을 마운트. 키 이름에 따라 각 항목의 파일이 생성됨
- 2. 환경 변수 : 환경변수의 값을 동적으로 설정 가능
- 3. 명령줄 인수 : configmap을 기반으로 컨테이너의 명령어 인수를 동적으로 변경

## 12. deployment
- Deployment는 ReplicaSet을 내부적으로 관리한다.
- Deployment가 ReplicaSet를 선택하는 방식은 label selector를 이용.
-

```
> kubectl run nginx --image=nginx:1.7.12
deployment "nginx" created

> kubectl get deployments nginx
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     1         1         1            1           11s

# Deployment가 run=nginx라는 ReplicaSet를 관리하는 것을 확인
> kubectl get deployments nginx -o jsonpath --template {.spec.selector.matchLabels}
map[run:nginx]

# label selector를 이용하여, 해당 ReplicaSet 확인
> kubectl get replicasets --selector=run=nginx
NAME               DESIRED   CURRENT   READY     AGE
nginx-7d9bf79f9b   1         1         1         3m

# Deployment의 복제본(스케일) 크기 조절
> kubectl scale deployments  nginx --replicas=2
deployment "nginx" scaled

> kubectl get replicasets --selector=run=nginx
NAME               DESIRED   CURRENT   READY     AGE
nginx-7d9bf79f9b   2         2         2         5m

# 여기서 주의할 사항!
# 만약 Deployment의 replicas가 2인데,
# ReplicaSet의 replicas를 1로 변경하면 어떻게 될까?
> kubectl scale rs nginx-7d9bf79f9b  --replicas=1
replicaset "nginx-7d9bf79f9b" scaled

# 1로 변경해도, 2로 계속 유지가 된다.
# 왜그럴까?
# -> 이미 상위 컨트롤러인 Deployment에서 2로 지정을 했기 떄문에,
# -> replicaSet이 1로 변경하려고 해도, Deployment가 2로 다시 조정하게 된다.
>  kubectl get replicasets --selector=run=nginx
NAME               DESIRED   CURRENT   READY     AGE
nginx-7d9bf79f9b   2         2         2         8m

```


- 동작중인 Deployment의 spec을 다운로드하고,
- 변경사항을 적용하는 방법
```
> kubectl get deployments nginx --export -o yaml > nginx-deployment.yaml
> kubectl replace -f nginx-deployment.yaml --save-config
deployment "nginx" replaced
```


### Deployment rollout
- nginx 버전을 업데이트하도록 이미지 명을 수정하고,
- rollout을 실행

```
> kubectl apply -f nginx-deployment-2.yaml
deployment "nginx" configured

> kubectl rollout status deployments nginx
deployment "nginx" successfully rolled out

> kubectl get replicasets -o wide
NAME               DESIRED   CURRENT   READY     AGE       CONTAINERS   IMAGES         SELECTOR
nginx-5dcc5d74d    2         2         2         37s       nginx        nginx:1.9.10   pod-template-hash=187718308,run=nginx
nginx-7d9bf79f9b   0         0         0         17m       nginx        nginx:1.7.12   pod-template-hash=3856935956,run=nginx

> kubectl rollout pause deployments nginx
deployment "nginx" paused

> kubectl rollout resume deployments nginx
deployment "nginx" resumed

> kubectl rollout history deployments nginx
deployments "nginx"
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

# 이전 버전으로 roll back 해 보자.
> kubectl rollout undo deployments nginx --to-revision=1
deployment "nginx"

> kubectl rollout history deployments nginx
deployments "nginx"
REVISION  CHANGE-CAUSE
2         <none>
3         <none>

```

### Deployment 전략
#### 롤링 업데이트 설정
- maxUnavailable : 롤링 업데이트 동안 사용하지 못하는 포드의 수.
  - 기존에 10개가 구동중이 었다면, 5로 지정하면, 5개만 서비스가 가능하고,
  - 나머지 5개의 포드에 업데이트를 진행하는 방식
  - 이 값이 크면, 한번에 업데이트 되는 포드가 많이지게 되어, 업데이트 속도가 빠르다.
- maxSurge : 롤아웃시 추가될 자원의 양을 조정
  - 10개의 복제본을 가진 서비스가 있을 경우,
  - maxUnavailable을 0으로 설정하고,
  - maxSurge를 20%로 설정하면,
  - 롤아웃시 2개의 복제본을 더 만들어서 replicaset를 총 12개(120%)로 만든다.
  - 다음으로 이전 replicset의 포드를 2개 줄여서 8개로 만들고, 새로운 복제본 2개로 총 10개로 만든다.
  - 이런 과정을 롤아웃이 완료될 때 까지 진행.
  - 이 방식은 언제나 서비스 용량을 100% 유지 가능한 장점 (추가 자원만 지원)

#### 안정적인 서비스를 위한 느린 롤아웃
### Rolling Update 방식
- 각 DaemonSet의 어플리케이션을 중단없이 업데이트 하려고 할때 사용함.
- spec.minReadySeconds : 다음 포드를 업데이트 하기전, 포드가 준비(ready) 상태로 있어야 하는 최소시간
  - 가끔 포드가 준비되었다고 해도, 실제로 정상동작하지 않는 경우가 발생
  - 심각한 메모리 누수일수도 있고, 일시적인 부하일 수도 있다.
  - 이런 경우를 대비해서, 업데이트된 pod가 정상상태가 될때 까지 대기하는 시간을 정의한다.
  - 60으로 설정된 경우, 포드가 정상상태임을 확인하고, 다음 포드를 업데이트하기 전 60초 동안 대기
  - 주의사항!
    - 이 지표는 포드가 ready 상태가 된 이후에 대기하는 시간이다.
    - 즉, 포드가 ready상태가 되지 않으면, ready 상태가 될 때 까지 무한정 기다리는 문제가 발생 (특별한 상황)
    - 따라서 Deployment의 전체 대기시간을 지정하여,
    - rolling update에 타임아웃을 설정하는 것이 필요하다.
    - spec.progressDeadlineSeconds: 600 진행 데드라인 설정.
      - 특정 단계가 10분 내에 진행되지 않으면, Deployment를 실패로 표시.
      - 이후 진행될 모든 시도가 중지됨.
- spec.updateStrategy.RollingUpdate.maxUnavailable : 롤링 업데이트시 동시에 업데이트 될 포드의 수 지정
  * 동시에 여러개를 업데이트 하면, 업데이트 속도는 빨라짐. (하지만 업데이트가 실패하면, 전체 서비스에 영향)


## 13. 스토리지
### 1. 외부 스토리지 활용하기
- kubernetes에서 외부 스토리지(DB, S3 등)에 접근할 수 있다.
- 문제는 외부 스토리지를 사용하는 포드들이 개발/운영 환경에서 동일한 설정으로 동작하도록 하는 것.
- 이를 위해 kubernetes 내부 서비스를 생성하고,
- 해당 서비스에서 개발환경에서는 내부주소, 운영환경에서는 외부 주소를 바라보도록 설정
- 결국 스토리지에 연결된 포드는 개발/운영 환경에서 모두 동일한 서비스만 바라볼 수 있다.
#### labelSelector 없는 서비스 생성
- 외부 스토리지를 바라보는 서비스는
- 내부 포드가 필요없다. 따라서 해당 외부 서비스 url만 정의 (externalName)
- Pod에서는 external-database에 접속하여, 외부 스토리지(db)에 연결하도록 설정하면,
- 개발환경과 운영환경이 모두 동일하게 구성될 수 있다.
- 주의사항!
  - kubernetes는 서비스에 포함된 포드의 상태를 체크해 주지만,
  - DNS로 정의된 서비스는 외부 DB의 상태를 확인할 수 없음. (운영자가 수동으로 확인)

##### DNS로 연결
```yaml

kind: Service
apiVersion: v1
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: "database.company.com"
```

##### IP로 연결
- 빈 서비스만 생성
```yaml
kind: Service
apiVersion: v1
metadata:
  name: external-ip-database
```

- 위에 생성한 서비스에 endpoint를 연결
```yaml
kind: Endpoints
apiVersion: v1
metadata:
  name: external-ip-database
subsets:
  - addresses:
    - ip: 192.168.0.1
    ports:
    - port: 3306
```

### 2. 신뢰 가능한 싱글톤 방식
- 복제본을 가지지 않는 단일 포드로 운영하는 방식 (아래 3개의 구성을 이용)
- 영구볼륨
- MySQL POD
- Service


### 3. StatefulSet
- 각 복제본은 고유한 인덱스와 영구 호스트네임을 가짐
- 각 복제본은 인덱스 번호가 낮은순에서 높은순으로 생성
- 삭제시에는 높은 순에서 낮은 순으로 생성
- StatefulSet을 사용하여 mongodb를 배포해 보자
```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 3
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongodb
        image: mongo:3.4.1
        command:
        - mongod
        - --replSet
        - rs0
        ports:
        - containerPort: 27017
          name: peer
```
```
> kubectl apply -f 13-10-mongo-simple.yaml
statefulset "mongo" created

# pod 명 뒤에 숫자로 된 index가 있음.
> kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
mongo-0   1/1       Running   0          5m
mongo-1   1/1       Running   0          4m
mongo-2   1/1       Running   0          4m
```

- Headless 서비스 생성 (클러스터에서 가상IP가 없는 서비스)
- StatefulSet은 고유한 독자성이 보장되야 함으로, 복제 서비스를 위한 로드밸런싱IP가 필요없다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  ports:
  - port: 27017
    name: peer
  clusterIP: None
  selector:
    app: mongo
```

- 서비스를 생성하면, StatefulSet의 모든 주소가 반환된다.
- 여기서는 mongo-1.mongo, mongo-2.mongo 등의 영구적인 host 명이 생성된다.  
```
> kubectl apply -f 13-11-mongo-service.yaml
```

- 이제 statefulset의 복제본간의 통신은 mongo-1.mongo 이름으로 통신이 가능하다.
- 아래 명령어로 확인 (실제는 에러 발생)
```
> kubectl exec mongo-0 bash ping mongo-1.mongo
```

#### Mongo 클러스터 생성
```
> kubectl apply -f 13-12-mongo-configmap.yaml
configmap "mongo-init" created

>  kubectl apply -f 13-11-mongo-service.yaml
service "mongo" created

> kubectl apply -f 13-13-mongo.yaml
statefulset "mongo" created

> kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
mongo-0   2/2       Running   0          47s
mongo-1   2/2       Running   0          32s
mongo-2   2/2       Running   0          16s
```

## 4. 실제 어플리케이션 배포
- mongodb cluster는 이미 설치되어 있다고 가정한다.
- 위의 스크립트로 설치 가능.
### 1). Parse server 배표


```
> git clone https://github.com/ParsePlatform/parse-server
> cd parse-server/

> sudo docker build -t freepsw/parse-server .
> sudo docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: freepsw
Password:
Login Succeeded

> sudo docker push freepsw/parse-server
```
- parse-server를 구동하는 deployment
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: parse-server
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        run: parse-server
    spec:
      containers:
      - name: parse-server
        image: freepsw/parse-server
        env:
        - name: PARSE_SERVER_DATABASE_URI
          value: "mongodb://mongo-0.mongo:27017,\
            mongo-1.mongo:27017,mongo-2.mongo\
            :27017/dev?replicaSet=rs0"
        - name: PARSE_SERVER_APP_ID
          value: my-app-id
        - name: PARSE_SERVER_MASTER_KEY
          value: my-master-key
```

```
> kubectl apply -f 14-1-parse.yaml
```
##### 설치 중 env를 정상적으로 전달하지 못하여 오류 발생 --> 원인 확인 필요


curl -X POST \
-H "X-Parse-Application-Id: my-app-id" \
-H "Content-Type: application/json" \
-d '{"score":1337,"playerName":"Sean Plott","cheatMode":false}' \
http://localhost:1337/parse/classes/GameScore





## 2) Resis cluster 배포


```
> kubectl create configmap --from-file=slave.conf=./14-5-slave.conf --from-file=master.conf=./14-4-master.conf --from-file=sentinel.conf=./14-6-sentinel.conf  --from-file=init.sh=./14-7-init.sh  --from-file=sentinel.sh=./14-8-sentinel.sh  redis-config

> kubectl  describe cm redis-config
Name:         redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
iㅏㅕ
----
#!/bin/bash
if [[ ${HOSTNAME} == 'redis-0' ]]; then
  redis-server /redis-config/master.conf
else
  redis-server /redis-config/slave.conf
fi

master.conf:
----
bind 0.0.0.0
port 6379

dir /redis-data

sentinel.conf:
----
bind 0.0.0.0
port 26379

sentinel monitor redis redis-0.redis 6379 2
sentinel parallel-syncs redis 1
sentinel down-after-milliseconds redis 10000
sentinel failover-timeout redis 20000

sentinel.sh:
----
#!/bin/bash
while ! ping -c 1 redis-0.redis; do
    echo 'Waiting for server'
    sleep 1
done

redis-sentinel /redis-config/sentinel.conf


slave.conf:
----
bind 0.0.0.0
port 6379

dir .

slaveof redis-0.redis 6379

Events:  <none>
```

```
> kubectl apply -f 14-9-redis-service.yaml

# 아래믜 명령을 실행하면, 에러가 발생한다.
> kubectl apply -f 14-10-redis.yaml

# 발생 로그 확인
# sentinel을 구동하는 container에서 sentinel.conf 파일을 수정하려는데,
# 읽기 권한만 부여된 상태라 수정이 불가능하여 오류 발생함.
# kubernetes 1.9.4이하 버전에서는 잘 동작한 것 같은데,
# 1.10 이상에서 발생하는 오류로 판단됨. ( https://tutel.me/c/programming/questions/49397309/unable+to+write+to+redisconf+pods+crashing)
# 1.10 이상의 샘플을 찾아서 해결하는 걸로...
> kubectl logs redis-0 -c sentinel
Sentinel config file /redis-config/sentinel.conf is not writable: Read-only file system. Exiting..


# 14-9-redis-service.yaml의 default mode를 666으로 변경시
# statefulset 확인
# 맨 아래에 defaultMode가 잘못된 범위로 입력되었다는 오류가 발생했다.
> kubectl describe statefulsets redis
Name:               redis
Namespace:          default
CreationTimestamp:  Mon, 13 Aug 2018 04:48:36 +0000
Selector:           app=redis
Labels:             app=redis
Annotations:        kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"apps/v1beta1","kind":"StatefulSet","metadata":{"annotations":{},"name":"redis","namespace":"default"},"spec":{"replicas":3,"serviceName"...
Replicas:           3 desired | 0 total
Pods Status:        0 Running / 0 Waiting / 0 Succeeded / 0 Failed
apiVersion: apps/v1beta1
Pod Template:
  Labels:  app=redis
  Containers:
   redis:
    Image:  redis:3.2.7-alpine
    Port:   6379/TCP
    Command:
      sh
      -c
      source /redis-config/init.sh
    Environment:  <none>
    Mounts:
      /redis-config from config (rw)
      /redis-data from data (rw)
   sentinel:
    Image:  redis:3.2.7-alpine
    Port:   <none>
    Command:
      sh
      -c
      source /redis-config/sentinel.sh
    Environment:  <none>
    Mounts:
      /redis-config from config (rw)
  Volumes:
   config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      redis-config
    Optional:  false
   data:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
Volume Claims:  <none>
Events:
  Type     Reason        Age                 From                    Message
  ----     ------        ----                ----                    -------
  Warning  FailedCreate  21s (x14 over 42s)  statefulset-controller  create Pod redis-0 in StatefulSet redis failed error: Pod "redis-0" is invalid: [spec.volumes[0].configMap.defaultMode: Invalid value: 666: must be a number between 0 and 0777 (octal), both inclusive, spec.containers[0].volumeMounts[0].name: Not found: "config", spec.containers[1].volumeMounts[0].name: Not found: "config"]
```
