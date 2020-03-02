# Mastering Kubernetes 2/e

## 1. Architecture

### API
- /api/v1
- /api/v2alpha
- API 그룹 활성/비활성화 (API 서버 실행시 옵션 추가)
  - --runtime-config=batch/v1=false,batch/v2alpha=true


## 2. 클러스터 생성하기

### Google Cloud에서 가상화 옵션을 지원하는 compute engine image 만들기
- https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances
```
> gcloud compute disks create disk1 --image-project centos-cloud --image-family centos-7
> gcloud components update
> gcloud compute images create nested-vm-image \
  --source-disk disk1 --source-disk-zone asia-northeast1-c \
  --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx"

> gcloud compute instances create example-nested-vm --zone asia-northeast1-c \
  --image nested-vm-image

> gcloud compute ssh example-nested-vm

# 가상화가 적용되었는지 확인 (1이 나오면 적용된 것)
> grep -cw vmx /proc/cpuinfo
```

### 2.1 Minikube (localhost 환경에 구성)
- MacOS
#### Install VirtualBox
- http://www.itzgeek.com/how-tos/linux/centos-how-tos/install-virtualbox-4-3-on-centos-7-rhel-7.html

```
> sudo yum -y install kernel-devel kernel-headers dkms
> sudo yum -y groupinstall "Development Tools"
> sudo yum update
> wget http://download.virtualbox.org/virtualbox/debian/oracle_vbox.asc
> sudo rpm --import oracle_vbox.asc
> sudo wget http://download.virtualbox.org/virtualbox/rpm/el/virtualbox.repo -O /etc/yum.repos.d/virtualbox.repo
> sudo yum -y install VirtualBox-5.1
```


#### Install kubectl
- https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-binary-via-curl
```
> curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
> chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

#### Install Minikube
https://kubernetes.io/docs/tasks/tools/install-minikube/

```
> curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.28.2/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/


```

### Interacting with my cluster
```

> minikube start

# minikube는 자동으로 minikube라는 context를 생성하고, 거기에 배포한다.
# 따라서 해당 context를 지정하거나, default context를 설정하도록 한다
> kubectl config use-context minikube
# 또는 매번 ocontext 지정
> kubectl get pods --context=minikube

```




## 3. monitoring
### 3.1 Monitoring in minikube
- https://github.com/kubernetes/minikube/blob/master/README.md

#### kubernetes dashboard
- 아래 명령을 실행하면, browser에 kubernetes dashboard가 보인다.
- 예시 : http://192.168.99.100:30000/#!/overview?namespace=default
```
> minikube dashboard
```

#### heapster (grafana) dashboard

##### Install heapster
- github에서 관련 코드를 다운받아 설치한다.
- https://github.com/kubernetes/heapster
-

```
> git clone https://github.com/kubernetes/heapster.git
> cd deploy/kube-config/influxdb/

# 해당 폴더에 있는 yaml 파일이 모니터링에 필요한 pod & service를 설치한다.
# 이 중 Service의 기본 정책은 cluster IP(내부)만 사용하지만,
# minikube는 localhost에서 외부로 노출할 수 있도록 type: NodePort를 추가한다.
# 그리고, 아래와 같이 배포 실행
> kubectl apply -f deploy/kube-config/influxdb/

# kube-system namespace에 생성함.
> kubectl describe service monitoring-influxdb --namespace=kube-system | grep NodePort
NodePort:                 <unset>  31578/TCP
```

##### Access to grafana dashboard
```
# minikube에서는 heapster addon이 디폴트로 disable됨. 이를 enable로 변경
> minikube addons enable heapster
heapster was successfully enabled

> minikube addons list
- addon-manager: enabled
- coredns: disabled
- dashboard: enabled
- default-storageclass: enabled
- efk: disabled
- freshpod: disabled
- heapster: enabled


> minikube service list
|-------------|----------------------|-----------------------------|
|  NAMESPACE  |         NAME         |             URL             |
|-------------|----------------------|-----------------------------|
| default     | hello-minikube       | http://192.168.99.100:31509 |
| default     | kubernetes           | No node port                |
| kube-system | heapster             | No node port                |
| kube-system | kube-dns             | No node port                |
| kube-system | kubernetes-dashboard | http://192.168.99.100:30000 |
| kube-system | monitoring-grafana   | No node port                |
| kube-system | monitoring-influxdb  | http://192.168.99.100:32618 |
|-------------|----------------------|-----------------------------|

# NodePort로 지정된 service는 "URL"이 보이게 되고,
# 웹 브라우저에서 해당 url로 접속할 수 있다.
```
- 그런데 실제 monitoring-grafana 서비스를 제외하고,
- heapster, monitoring-influxdb는 404 page not found 에러가 발생한다.
- 그래서 kubectl cluster-info에서 제공하는 url로 접속하면 또 다른 에러 발생
```
> kubectl cluster-info
Kubernetes master is running at https://192.168.99.100:8443
Heapster is running at https://192.168.99.100:8443/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://192.168.99.100:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
monitoring-grafana is running at https://192.168.99.100:8443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
monitoring-influxdb is running at https://192.168.99.100:8443/api/v1/namespaces/kube-system/services/monitoring-influxdb/proxy
```
- 웹 브라우저에 표시되는 메세지
```json
{
  kind: "Status",
  apiVersion: "v1",
  metadata: { },
  status: "Failure",
  message: "services "monitoring-influxdb" is forbidden: User "system:anonymous" cannot get services/proxy in the namespace "kube-system"",
  reason: "Forbidden",
  details: {
  name: "monitoring-influxdb",
  kind: "services",
  },
  code: 403,
}
```
- 일단은 정확한 원인을 알 수 없어서 패스
- kubectl apply -f rbac/heapster-rbac.yaml -n kube-syste 를 적용해도 동일한 문제 발생



- heapster 사용 사례
- https://dzone.com/articles/how-to-utilize-the-heapster-influxdb-grafana-stack

```
> minikube deleteb
```


### 3.2 Prometheus 활용한 모니터링

```
> git clone https://github.com/coreos/prometheus-operator.git

> cd prometheus-operator/contrib/kube-prometheus

> kubectl create -f manifests/ || true

> hack/example-service-monitoring/deploy
service/example-app unchanged
deployment.extensions/example-app unchanged
rolebinding.rbac.authorization.k8s.io/prometheus-frontend created
role.rbac.authorization.k8s.io/alertmanager-discovery created
rolebinding.rbac.authorization.k8s.io/prometheus-frontend unchanged
role.rbac.authorization.k8s.io/prometheus-frontend unchanged
serviceaccount/prometheus-frontend unchanged
service/prometheus-frontend unchanged
prometheus.monitoring.coreos.com/frontend created
servicemonitor.monitoring.coreos.com/frontend created

> kubectl get pod -n monitoring
NAME                                  READY     STATUS              RESTARTS   AGE
alertmanager-main-0                   0/2       ContainerCreating   0          2m
grafana-5b68464b84-k5tzq              0/1       ContainerCreating   0          2m
kube-state-metrics-64df5b9d5c-tdw74   0/4       ContainerCreating   0          2m
node-exporter-n2qmr                   0/2       ContainerCreating   0          2m
prometheus-k8s-0                      0/3       ContainerCreating   0          2m
prometheus-operator-9b4b64465-q8tk8   1/1       Running             0          2m
```
- 정상 작동하지 않음.... 확인 필요

### 3.3 Node problem detector
- https://github.com/kubernetes/node-problem-detector
#### 하드웨어 장애
##### 노드의 응답이 없는 경우
- 응답이 없으므로 구체적인 원인을 찾기 어려움
- 기존 응답을 통해 유추하거나,
- 새로 추가된 노드의 경우 설정의 문제일 가능성이 높음
##### 노드의 응답이 있는 경우
- 디스크나 코어 같은 하드웨어 장애 가능성 높음
- 포드의 지속적 재시작 및 포드 시작시간 증가 등


## 4. 고가용성과 신뢰성
### 4.1 고가용성이란?
- 중복성 : 장애발생 시 즉시 중복된 구성요소가 전체 서비스의 중단없이 시스템을 유지할 수 있다.
- Hot Swapping : 장애가 발생한 구성요소를 즉시 교체(무중단 서비스)하는 개념
- 리더 선출 : 모든 구성요소가 리더가 될 수 있도록 구성
- 스마트 로드밸런싱 : 요청된 서비스를 여러 구성요소에 분배함 (장애 및 미응답 구성요소는 요청 전달 중지)
- 멱등성 : 장애로 인하여 처리 중인 작업이 다른 구성요소에서 실행되더라도 동일한 결과값이 생성되도록 해야 함
- 자가치유 : 문제 발생시 자동감시와 자동해결을 통해 노드의 상태를 유지
