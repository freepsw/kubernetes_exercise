# Getting Started with Kubernetes - second edition (쿠버네티스 기초다지기)
kubernetes 관련 작업을 정리 (사례 중심)
-https://gist.github.com/edsiper/fac9a816898e16fc0036f5508320e8b4 참고

- Google kubernetes 인증
  * gcloud container clusters get-credentials kuar-cluster --zone asia-northeast1-c --project myproject-209707


## 1. GCP에 kubernetes cluster 구성하기
- https://kubernetes.io/docs/setup/turnkey/gce/

```

> curl https://sdk.cloud.google.com | bash

# SDK를 인증할 수 있도록 인증코드를 입력한다.
> gcloud auth login --no-launch-browser

> cloud config list project
[core]
project = myproject-209707

Your active configuration is: [default]

> gcloud config set project myproject-209707
Updated property [core/project].

> curl -sS https://get.k8s.io | bash

# 현재 디렉토리 아래에 kubernetes 폴더가 생성됨.
> kubernetes/cluster/kube-up.sh

# 아래와 같은 오류 발생시 해당 component 설치
missing required gcloud component "alpha"
missing required gcloud component "beta"

> gcloud components install alpha
> gcloud components install beta


# 필요한 컴포넌트 설치 후, 다시 실행
# 설치가 끝나면, 아래의 메세지가 보인다.
  # master ip 정보  : https://35.232.112.154
  # 사용자 정보가 저장된 파일 경 : /home/freepsw_02/.kube/config.
  # GCE의 Project 정보 : project id, zone, 구성요소 상태정보
  # Kubernetes 서비스 URL 정보
> kubernetes/cluster/kube-up.sh
....................................
Kubernetes cluster created.
Cluster "myproject-209707_kubernetes" set.
User "myproject-209707_kubernetes" set.
Context "myproject-209707_kubernetes" created.
Switched to context "myproject-209707_kubernetes".
User "myproject-209707_kubernetes-basic-auth" set.
Wrote config for myproject-209707_kubernetes to /home/freepsw_02/.kube/config


Kubernetes cluster is running.  The master is running at:
  https://35.224.192.64

The user name and password to use is located in /home/freepsw_02/.kube/config.

... calling validate-cluster
Validating gce cluster, MULTIZONE=
Project: myproject-209707
Network Project: myproject-209707
Zone: us-central1-b
Found 4 node(s).
NAME                           STATUS                     ROLES     AGE       VERSION
kubernetes-master              Ready,SchedulingDisabled   <none>    9m        v1.11.2
kubernetes-minion-group-f46r   Ready                      <none>    9m        v1.11.2
kubernetes-minion-group-qhmb   Ready                      <none>    9m        v1.11.2
kubernetes-minion-group-zbxp   Ready                      <none>    9m        v1.11.2
Validate output:
NAME                 STATUS    MESSAGE              ERROR
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
Cluster validation succeeded
Done, listing cluster services:



Kubernetes master is running at https://35.224.192.64
GLBCDefaultBackend is running at https://35.224.192.64/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
Heapster is running at https://35.224.192.64/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://35.224.192.64/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://35.224.192.64/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
Metrics-server is running at https://35.224.192.64/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

```


### Kubernetes UI에 접속하기 위해서는 별도의 권한이 필요
```
> kubectl create clusterrolebinding cluster-system-anonymous --clusterrole=cluster-admin --user=system:anonymous
```

### 초기 사용자 인증에 필요한 token 입력
- dashboard에 처음 접속하면, 인증을 위한 설정이 필요함.
- token값은 kubectl config view로 확인 가능
```
> kubectl config view
apiVersion: v1
clusters:
................
- name: myproject-209707_kubernetes
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
    token: 1qvrkrVtkY4eziHdtKE6pzXPyYSE9V9d  --> 사용자 토큰값
- name: myproject-209707_kubernetes-basic-auth
  user:
    password: aUo5jLPQ4rX0D1nl
    username: admin
```

```
> kubectl cluster-info
Kubernetes master is running at https://35.224.192.64
GLBCDefaultBackend is running at https://35.224.192.64/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
Heapster is running at https://35.224.192.64/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://35.224.192.64/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://35.224.192.64/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
Metrics-server is running at https://35.224.192.64/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

```
> kubectl get pods --namespace=kube-system
NAME                                        READY     STATUS    RESTARTS   AGE
etcd-empty-dir-cleanup-kubernetes-master    1/1       Running   0          1h
etcd-server-events-kubernetes-master        1/1       Running   0          1h
etcd-server-kubernetes-master               1/1       Running   0          1h
event-exporter-v0.2.1-c6b669bc7-8xtcq       1/1       Running   0          1h
fluentd-gcp-scaler-5d85d4b48b-9nk5n         1/1       Running   0          1h
fluentd-gcp-v3.1.0-6drc7                    1/1       Running   0          1h
fluentd-gcp-v3.1.0-hkwfz                    1/1       Running   0          1h
fluentd-gcp-v3.1.0-nqrks                    1/1       Running   0          1h
fluentd-gcp-v3.1.0-sxwwt                    1/1       Running   0          1h
heapster-v1.5.3-5d976b745f-xsfb5            2/2       Running   0          1h
kube-addon-manager-kubernetes-master        1/1       Running   0          1h
kube-apiserver-kubernetes-master            1/1       Running   0          1h
kube-controller-manager-kubernetes-master   1/1       Running   0          1h
kube-dns-7b479ccbc6-944xp                   3/3       Running   0          1h
kube-dns-7b479ccbc6-ddb2l                   3/3       Running   0          1h
kube-dns-autoscaler-67c97c87fb-7vdlj        1/1       Running   0          1h
kube-proxy-kubernetes-minion-group-8f0w     1/1       Running   0          1h
kube-proxy-kubernetes-minion-group-cv6s     1/1       Running   0          1h
kube-proxy-kubernetes-minion-group-n7ct     1/1       Running   0          1h
kube-scheduler-kubernetes-master            1/1       Running   0          1h
kubernetes-dashboard-db894757f-jf2ws        1/1       Running   0          1h
l7-default-backend-5bc54cfb57-b22ns         1/1       Running   0          1h
l7-lb-controller-v1.1.1-kubernetes-master   1/1       Running   2          1h
metrics-server-v0.2.1-fd596d746-clvx5       2/2       Running   0          1h
rescheduler-v0.4.0-kubernetes-master        1/1       Running   0          1h
```

### cluster 삭제
```
> cluster/kube-down.sh
```

## 4. rolling update
### scale command를 이용하여 직접 rolling update 실행
- 최신 버전에서는 Deployment를 통해 rolling update를 하는 것이 효과적임.
```
> kubectl apply -f pod-scaling-controller.yaml
> kubectl apply -f pod-scaling-service.yaml
> kubectl get pods
NAME                  READY     STATUS    RESTARTS   AGE
node-js-scale-6bd2z   1/1       Running   0          34m
node-js-scale-mzgsz   1/1       Running   0          34m
node-js-scale-ntcrw   1/1       Running   0          39m

> rolling-update node-js-scale node-js-scale-v2.0 --image=jonbaier/pod-scaling:0.2 --update-period="2m"
Command "rolling-update" is deprecated, use "rollout" instead
Created node-js-scale-a3b87ba56900068867550a60a885b660
Scaling up node-js-scale-a3b87ba56900068867550a60a885b660 from 0 to 3, scaling down node-js-scale from 3 to 0 (keep 3 pods available, don't exceed 4 pods)
Scaling node-js-scale-a3b87ba56900068867550a60a885b660 up to 1
```

## 5. deployment
- ReplicationController와 다른 점은 Deployment 오브젝트를 변경 및 업데이트하여,
- kubernetes가 pod와 replica를 업데이트 하도록 관리할 수 있다.
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: node-js-deploy
  labels:
    name: node-js-deploy
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: node-js-deploy
    spec:
      containers:
      - name: node-js-deploy
        image: jonbaier/pod-scaling:0.1
        ports:
        - containerPort: 80
```

- deployment를 배포하자
- 이떄 record 옵션을 사용하면, deployment 생성이 rollout history에 기록된다.
```
> kubectl apply -f node-js-deploy.yaml --record
deployment.extensions/node-js-deploy created

> get pod
NAME                             READY     STATUS    RESTARTS   AGE
node-js-deploy-fc5f5fc79-r5ws2   1/1       Running   0          21s
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-js-deploy
  labels:
    name: node-js-deploy
spec:
  type: LoadBalancer
  ports:
  - port: 80
  sessionAffinity: ClientIP
  selector:
    name: node-js-deploy
```

```
> kubectl apply -f node-js-deploy-service.yaml
service/node-js-deploy created
```

- Scale 확장
```
> kubectl scale deployment node-js-deploy --replicas 3
deployment.extensions/node-js-deploy scaled

> kubectl get pod
NAME                             READY     STATUS    RESTARTS   AGE
node-js-deploy-fc5f5fc79-95wsn   1/1       Running   0          5s
node-js-deploy-fc5f5fc79-nt7pv   1/1       Running   0          5s
node-js-deploy-fc5f5fc79-r5ws2   1/1       Running   0          5m
```

- container image version 확인 후,
- 2.0으로 업그레이드
```
> kubectl describe pod node-js-deploy-fc5f5fc79-95wsn | grep Image
    Image:          jonbaier/pod-scaling:0.1

> kubectl set image deployment/node-js-deploy node-js-deploy=jonbaier/pod-scaling:0.2
deployment.extensions/node-js-deploy image updated

# 아래와 같이 pod가 업그레드 되고 있다.
> kubectl get pod
NAME                              READY     STATUS              RESTARTS   AGE
node-js-deploy-5865d4f55d-dlnnb   1/1       Running             0          5s
node-js-deploy-5865d4f55d-kn6r9   1/1       Running             0          5s
node-js-deploy-5865d4f55d-r96nn   0/1       ContainerCreating   0          3s
node-js-deploy-fc5f5fc79-95wsn    1/1       Terminating         0          5m
node-js-deploy-fc5f5fc79-nt7pv    1/1       Terminating         0          5m
node-js-deploy-fc5f5fc79-r5ws2    1/1       Terminating         0          10m

# rollout 결과 확인
> kubectl rollout status deployment/node-js-deploy
deployment "node-js-deploy" successfully rolled out
```

### Rollout history
- rollout을 실행한 이력확인
```
> kubectl rollout history deployment/node-js-deploy
deployments "node-js-deploy"
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

### Rollout undo
- 업데이트가 진행중인 상태에서, 이전버전으로 다시 롤백 가능
```
> kubectl rollout undo deployment/node-js-deploy
deployment.extensions/node-js-deploy

> kubectl get pod
NAME                              READY     STATUS        RESTARTS   AGE
node-js-deploy-5865d4f55d-2zjvd   1/1       Running       0          7s
node-js-deploy-5865d4f55d-mjcm7   1/1       Running       0          5s
node-js-deploy-5865d4f55d-ptgnd   1/1       Running       0          7s
node-js-deploy-d86db9785-7wwvj    1/1       Terminating   0          3m
node-js-deploy-d86db9785-gkjsb    1/1       Terminating   0          3m
node-js-deploy-d86db9785-wbcc8    1/1       Terminating   0          3m
[freepsw_02@k-client Chapter05]$
```

### Auto-scaling (HorizontalPodAutoscaler) - HPA
- 사용자가 지정한 제약을 넘어서는 경우,
- 자동으로 pod를 확장한다.
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: node-js-deploy
spec:
  minReplicas: 3
  maxReplicas: 6
  scaleTargetRef:
    apiVersion: v1
    kind: Deployment
    name: node-js-deploy
  targetCPUUtilizationPercentage: 10
```
