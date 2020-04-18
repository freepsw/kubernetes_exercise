## Section 7: Continuous Development with Kubernetes

### Introduction to Flux
- application을 build/push/deploy(kubernetes로) 하는 workflow를 다루는 오픈소스  
- git에 정의된 kubernetes 리소스(.yaml 파일들) 상태와 운영중인 kubernetes 상태를 동일하게 유지하는 GitOps 기술
  - 즉, git의 파일들을 모니터링 하다가,
  - 변경이 발생하면 이를 kubernetes에 자동으로 반영해 준다.
  - 특히, container registry에 이미지 버전 변경이 생기면, 이를 kubernetes의 container에 반영해 주는 기능을 제공.
  - github.com/fluxcd/flux

### Code Examples 1
- 초기 flux 설치 후 git의 내용을 k8s에 반영
#### Install flux on macos
```
> brew install fluxctl

> kubectl create ns flux
namespace/flux created

> kubectl get ns
NAME                   STATUS   AGE
default                Active   105d
flux                   Active   6s
......
```

#### 실습 시나리오
```
0. git repository를 생성
1. flux 실행 (git repository를 지정)
  - flux에서 git에 접근 할 수 있도록 public key 생성
  - git repository에 flux의 public key 등록
2. flux에서 git에 정의된 리소스 생성 확인
  - flux에서 git의 정보를 정상적으로 가져와서 namespace, deployment, service를 생성하는지 확인  
```

#### 0) flux에서 접속할 수 있는 git repository 생성 및 yaml코드 등록
- github계정/flux-demo (실습으로 https://github.com/wardviaene/flux-demo 활용)


#### 1) k8s에 flux 실행
```
> export GHUSER="freepsw"

> fluxctl install \
--git-user=${GHUSER} \
--git-email=${GHUSER}@users.noreply.github.com \
--git-url=git@github.com:${GHUSER}/flux-demo \  # github에 flux-demo라는 repository를 대상으로 모니터링
--git-path=namespaces,workloads \  # helm은 사용하지 않음을 명시
--namespace=flux | kubectl apply -f -

secret/flux-git-deploy created
deployment.apps/memcached created
service/memcached created
serviceaccount/flux created
clusterrole.rbac.authorization.k8s.io/flux created
clusterrolebinding.rbac.authorization.k8s.io/flux created
deployment.apps/flux created

# flux가 정상적으로 실행되는지 확인
> kubectl -n flux rollout status deployment/flux
deployment "flux" successfully rolled out

> kubectl get pod -n flux
NAME                         READY   STATUS    RESTARTS   AGE
flux-7fb567c798-j4zl5        1/1     Running   0          16s
memcached-86bdf9f56b-2g7pc   1/1     Running   0          60m
>
```

#### 2) flux에서 git에 접속하기 위한 설정 (여기서는 github계정을 사용)
##### k8s의 flux가 github에 안전하게 접근할 수 있도록 key를 생성
- github의 secret key로 해석 가능한 public key를 생성하여
- github 설정에 추가한다.
```
> fluxctl identity --k8s-fwd-ns flux
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDf96Z1Jzk28L9gRd9xSAKAHxgLK1/9qBd......
```

##### github flux-demo repository의 설정에서 위의 생성된 키값을 등록
- flux-demo > Settings > Deploy Keys > "Add depoy key" button click >
- "Key" 필드에 위에서 생성한 key값 입력
- "Allow write access" 체크박스 클릭 (flux에서 repository 변경 가능)
- "Add key" 클릭


##### flux가 git에 정의된 상태를 k8s에 적용하는 과정 (log 상세)
```
> kubectl logs flux-7fb567c798-j4zl5 -n flux
ts=2020-04-18T03:45:22.245950785Z caller=main.go:259 version=1.19.0
ts=2020-04-18T03:45:22.246247787Z caller=main.go:412 msg="using kube config: \"/root/.kube/config\" to connect to the cluster"
ts=2020-04-18T03:45:22.27101224Z caller=main.go:492 component=cluster identity=/etc/fluxd/ssh/identity
ts=2020-04-18T03:45:22.271196664Z caller=main.go:493 component=cluster identity.pub="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDf96Z1Jzk28L9gR......."
ts=2020-04-18T03:45:22.271300726Z caller=main.go:498 host=https://10.96.0.1:443 version=kubernetes-v1.17.0
ts=2020-04-18T03:45:22.271396795Z caller=main.go:510 kubectl=/usr/local/bin/kubectl
ts=2020-04-18T03:45:22.272661158Z caller=main.go:527 ping=true
ts=2020-04-18T03:45:22.273867163Z caller=main.go:666 url=ssh://git@github.com/freepsw/flux-demo user=freepsw email=freepsw@users.noreply.github.com signing-key= verify-signatures-mode=none sync-tag=flux state=git readonly=false registry-disable-scanning=false notes-ref=flux set-author=false git-secret=false sops=false
ts=2020-04-18T03:45:22.273915187Z caller=main.go:772 upstream="no upstream URL given"
ts=2020-04-18T03:45:22.274035495Z caller=main.go:803 metrics-addr=:3031
ts=2020-04-18T03:45:22.274343807Z caller=loop.go:107 component=sync-loop err="git repo not ready: git repo has not been cloned yet"  # 아직 git에 연결되지 못함.
ts=2020-04-18T03:45:22.274423071Z caller=images.go:17 component=sync-loop msg="polling for new images for automated workloads"
ts=2020-04-18T03:45:22.274479915Z caller=images.go:27 component=sync-loop msg="no automated workloads"
ts=2020-04-18T03:45:22.277014771Z caller=main.go:795 addr=:3030
ts=2020-04-18T03:45:23.186062293Z caller=checkpoint.go:24 component=checkpoint msg="up to date" latest=1.19.0
ts=2020-04-18T03:45:39.320018514Z caller=loop.go:133 component=sync-loop event=refreshed url=ssh://git@github.com/freepsw/flux-demo branch=master # git에 연결하여 yaml파일을 가져옴.  HEAD=e29eb4d18200d51a0576f585814417cf97607ec0
ts=2020-04-18T03:45:39.338385915Z caller=sync.go:73 component=daemon info="trying to sync git changes to the cluster" old=e29eb4d18200d51a0576f585814417cf97607ec0  new=e29eb4d18200d51a0576f585814417cf97607ec0  # git의 상태를 k8s에 반영
ts=2020-04-18T03:45:39.558180375Z caller=sync.go:539 method=Sync cmd=apply args= count=3 # 총 3개의 yaml 파일을 생성
ts=2020-04-18T03:45:39.987909541Z caller=sync.go:605 method=Sync cmd="kubectl apply -f -" took=429.636106ms err=null output="namespace/demo created\nservice/docker-nodejs-demo created\ndeployment.apps/docker-nodejs-demo created"
ts=2020-04-18T03:45:44.914695261Z caller=loop.go:226 component=sync-loop state="tag flux" old=e29eb4d18200d51a0576f585814417cf97607ec0 new=e29eb4d18200d51a0576f585814417cf97607ec0
```

###### 여기서 의미있게 볼 log msg 확인
```
# 아직 git에 연결되지 못함.
caller=loop.go:107 component=sync-loop err="git repo not ready: git repo has not been cloned yet"  

# git에 연결하여 yaml파일을 가져옴.
caller=loop.go:133 component=sync-loop event=refreshed url=ssh://git@github.com/freepsw/flux-demo branch=master

# 총 3개의 yaml 파일을 k8s에 반영
caller=sync.go:539 method=Sync cmd=apply args= count=3 # 총 3개의 yaml 파일을 생성

# "kubectl apply -f -"명령으로 3개의 파일을 실행한 결과 출력
caller=sync.go:605 method=Sync cmd="kubectl apply -f -" took=429.636106ms err=null output="namespace/demo created\nservice/docker-nodejs-demo created\ndeployment.apps/docker-nodejs-demo created"
```

#### 3) 생성된 서비스 확인
```
> kubectl -n demo get pods
NAME                                  READY   STATUS    RESTARTS   AGE
docker-nodejs-demo-559b876f98-4zj2j   1/1     Running   0          24m

> kubectl -n demo get svc
NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
docker-nodejs-demo   NodePort   10.96.175.246   <none>        80:31624/TCP   25m

# 생성된 서비스에 접속해 보면, v1.0.2 웹 서비스로 접근
> curl $(minikube ip):31624
Hello World (v1.0.2)!
```

### Code Examples 2
- git repository에 변경 발생시
- 만약, container image를 v1.0.2에서 1.0.1로 변경하면?

#### 테스트 시나리오
```
1. github-user/flux-demo에서 pod의 이미지 버전 변경 (v1.0.2 --> 1.0.1로 rollback )
  - 그런데 k8s의 annotation에서 항상 v1.0.0이상의 최신버전을 사용하도록 정의했다면?
  - annotations:
    fluxcd.io/automated: "true"
    fluxcd.io/tag.docker-nodejs-demo: semver:~1.0.0 (semantic으로 1.0.0이상의 최신버전만 container repository에서 가져오도록 설정 )
2. github의 코드와 container repository의 버전이 다르면 어떻게 실행되는지 확인
  - 결과적으로 container image 버전을 최신으로 유지하기 위해서,
  - github의 코드를 변경(v1.0.1 -> v1.0.2)하여 commit (그래서 github 쓰기권한이 필요)
```

#### 1) github 코드 변경
- 아래 코드로 변경후 commit
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-nodejs-demo
  namespace: demo
  labels:
    app: docker-nodejs-demo
  annotations:
    fluxcd.io/automated: "true"
    fluxcd.io/tag.docker-nodejs-demo: semver:~1.0.0
spec:
  strategy:
    rollingUpdate:
      maxUnavailable: 0
    type: RollingUpdate
  selector:
    matchLabels:
      app: docker-nodejs-demo
  template:
    metadata:
      labels:
        app: docker-nodejs-demo
    spec:
      containers:
      - name: docker-nodejs-demo
        image: wardviaene/docker-nodejs-demo:1.0.1 # v1.0.2에서 v1.0.1로 rollback
        ports:
        - name: nodejs-port
          containerPort: 3000

```

#### 2) flux에서 변경된 코드 감지 후 kubernetes에 반영 준비
- 단계별로 중요한 로그를 보면, flux가 어떻게 동작하는지 이해할 수 있다.
```
# git의 코드에서 변경을 감지한다. (해당 코드가 )
caller=loop.go:133 component=sync-loop event=refreshed url=ssh://git@github.com/freepsw/flux-demo branch=master HEAD=ef5fc5c8cea1bdc581525cfb41fdf904089b634c
caller=images.go:17 component=sync-loop msg="polling for new images for automated workloads"

# 그런데 여기서 flux가 코드의 변경사항과 deployment의 규칙(최신 버전만 배포)를 비교하고,
# pod의 규칙을 벗어나는 것을 확인하는 것이 보인다. (reason="latest 1.0.2 (2020-01-23 12:08:04.619877884 +0000 UTC) > current 1.0.1 (2020-01-23 12:29:44.989057983 +0000 UTC)")
caller=images.go:111 component=sync-loop workload=demo:deployment/docker-nodejs-demo container=docker-nodejs-demo repo=wardviaene/docker-nodejs-demo pattern=semver:~1.0.0 current=wardviaene/docker-nodejs-demo:1.0.1 info="added update to automation run" new=wardviaene/docker-nodejs-demo:1.0.2 reason="latest 1.0.2 (2020-01-23 12:08:04.619877884 +0000 UTC) > current 1.0.1 (2020-01-23 12:29:44.989057983 +0000 UTC)"

# 현재 git의 코드에서 container 버전을 최신(v1.0.2)으로 변경하고,
# commit(Automated release of wardviaene/docker-nodejs-demo:1.0.2)한다.
caller=daemon.go:701 component=daemon event="Sync: 8acba00, demo:deployment/docker-nodejs-demo" logupstream=false
caller=daemon.go:701 component=daemon event="Automated release of wardviaene/docker-nodejs-demo:1.0.2" logupstream=false

```

- 전체 로그 확인
```
ts=2020-04-18T04:20:57.818131287Z caller=daemon.go:701 component=daemon event="Sync: ef5fc5c, demo:deployment/docker-nodejs-demo" logupstream=false
ts=2020-04-18T04:21:03.905530289Z caller=loop.go:226 component=sync-loop state="tag flux" old=e29eb4d18200d51a0576f585814417cf97607ec0 new=ef5fc5c8cea1bdc581525cfb41fdf904089b634c
ts=2020-04-18T04:21:06.343543387Z caller=loop.go:133 component=sync-loop event=refreshed url=ssh://git@github.com/freepsw/flux-demo branch=master HEAD=ef5fc5c8cea1bdc581525cfb41fdf904089b634c
ts=2020-04-18T04:21:41.751100295Z caller=images.go:17 component=sync-loop msg="polling for new images for automated workloads"
ts=2020-04-18T04:21:41.798682052Z caller=images.go:111 component=sync-loop workload=demo:deployment/docker-nodejs-demo container=docker-nodejs-demo repo=wardviaene/docker-nodejs-demo pattern=semver:~1.0.0 current=wardviaene/docker-nodejs-demo:1.0.1 info="added update to automation run" new=wardviaene/docker-nodejs-demo:1.0.2 reason="latest 1.0.2 (2020-01-23 12:08:04.619877884 +0000 UTC) > current 1.0.1 (2020-01-23 12:29:44.989057983 +0000 UTC)"
ts=2020-04-18T04:21:41.799014856Z caller=loop.go:141 component=sync-loop jobID=253431f4-4a2d-1448-208c-525f60517d73 state=in-progress
ts=2020-04-18T04:21:41.836590481Z caller=releaser.go:59 component=sync-loop jobID=253431f4-4a2d-1448-208c-525f60517d73 type=release updates=1
ts=2020-04-18T04:21:47.220534Z caller=daemon.go:292 component=sync-loop jobID=253431f4-4a2d-1448-208c-525f60517d73 revision=8acba00d526e8d70eb1b0adfcdf088c948fa7e3c
ts=2020-04-18T04:21:47.220608674Z caller=daemon.go:701 component=daemon event="Commit: 8acba00, demo:deployment/docker-nodejs-demo" logupstream=false
ts=2020-04-18T04:21:47.22425269Z caller=loop.go:153 component=sync-loop jobID=253431f4-4a2d-1448-208c-525f60517d73 state=done success=true
ts=2020-04-18T04:21:49.785566203Z caller=loop.go:133 component=sync-loop event=refreshed url=ssh://git@github.com/freepsw/flux-demo branch=master HEAD=8acba00d526e8d70eb1b0adfcdf088c948fa7e3c
ts=2020-04-18T04:21:49.799577739Z caller=sync.go:73 component=daemon info="trying to sync git changes to the cluster" old=ef5fc5c8cea1bdc581525cfb41fdf904089b634c new=8acba00d526e8d70eb1b0adfcdf088c948fa7e3c
ts=2020-04-18T04:21:49.928505973Z caller=sync.go:539 method=Sync cmd=apply args= count=3
ts=2020-04-18T04:21:50.097854596Z caller=sync.go:605 method=Sync cmd="kubectl apply -f -" took=169.069483ms err=null output="namespace/demo unchanged\nservice/docker-nodejs-demo unchanged\ndeployment.apps/docker-nodejs-demo configured"
ts=2020-04-18T04:21:50.115344638Z caller=daemon.go:701 component=daemon event="Sync: 8acba00, demo:deployment/docker-nodejs-demo" logupstream=false
ts=2020-04-18T04:21:50.115512292Z caller=daemon.go:701 component=daemon event="Automated release of wardviaene/docker-nodejs-demo:1.0.2" logupstream=false
ts=2020-04-18T04:21:55.193207602Z caller=loop.go:226 component=sync-loop state="tag flux" old=ef5fc5c8cea1bdc581525cfb41fdf904089b634c new=8acba00d526e8d70eb1b0adfcdf088c948fa7e3c
ts=2020-04-18T04:21:57.69320297Z caller=loop.go:133 component=sync-loop event=refreshed url=ssh://git@github.com/freepsw/flux-demo branch=master HEAD=8acba00d526e8d70eb1b0adfcdf088c948fa7e3c
```


### 3) 서비스 접속 (v1.0.1로 rollback 되지 않았음.)
```
> curl $(minikube ip):31624
Hello World (v1.0.2)!
```
