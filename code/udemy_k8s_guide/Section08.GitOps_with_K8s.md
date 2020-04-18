## Section 7: Continuous Development with Kubernetes

### Introduction to Flux
- application을 build/push/deploy(kubernetes로) 하는 workflow를 다루는 오픈소스  
- git에 정의된 kubernetes 리소스(.yaml 파일들) 상태와 운영중인 kubernetes 상태를 동일하게 유지하는 GitOps 기술
  - 즉, git의 파일들을 모니터링 하다가,
  - 변경이 발생하면 이를 kubernetes에 자동으로 반영해 준다.
  - 특히, container registry에 이미지 버전 변경이 생기면, 이를 kubernetes의 container에 반영해 주는 기능을 제공.
  - github.com/fluxcd/flux

### Code Examples
#### Install flux on macos
```
> brew install fluxctl
```

#### Configure flux env
```
> kubectl create ns flux
namespace/flux created

> kubectl get ns
NAME                   STATUS   AGE
default                Active   105d
flux                   Active   6s
......
```

# flux에서 git에 접속하기 위한 설정 (여기서는 github계정을 사용)
```
# 사전에 github 연결에 필요한 인증 필요
> export GHUSER="freepsw"
```

# flux 명령으로 git 연결에 필요한 정보 저장 (이후 flux에서 연동)
```
> fluxctl install \
--git-user=${GHUSER} \
--git-email=${GHUSER}@users.noreply.github.com \
--git-url=git@github.com:${GHUSER}/flux-demo \
--git-path=namespaces,workloads \
--namespace=flux | kubectl apply -f -
secret/flux-git-deploy created
deployment.apps/memcached created
service/memcached created
serviceaccount/flux created
clusterrole.rbac.authorization.k8s.io/flux created
clusterrolebinding.rbac.authorization.k8s.io/flux created
deployment.apps/flux created
```
