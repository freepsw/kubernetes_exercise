# Heml 이해
- https://docs.helm.sh/  참고

## 1. 구성요소
- Tiller 서버
  - 쿠버네테스 API와 통신하여 Helm 패키지를 관리하는 서비스.
  - Helm client에서 들어오는 요청 대기
  - 차트와 설정을 결합하여 배포판 빌드
  - 쿠버네티스에 차트 설치 및 업그레이드, 삭제

- Helm client
  - 차트 개발
  - 저장소 관리
  - Tiller 서버와 상호작용 및 차트전송
  - 배포한에 대한 정보 요청 (설치 해제 및 업그레이드)

## 2. Helm 설치
- https://docs.helm.sh/using_helm/#quickstart-guide
- INSTALLING THE HELM CLIENT
```
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

- INSTALLING TILLER
  - Tiller는 helm의 서버역햘을 수해하며, kubernetes 클러스터에서 실행된다.
  - tiller를 kubernetes 클러스터에 배포하는 방법은 helm init만 실행

```
> helm init
Creating /home/freepsw_02/.helm
Creating /home/freepsw_02/.helm/repository
Creating /home/freepsw_02/.helm/repository/cache
Creating /home/freepsw_02/.helm/repository/local
Creating /home/freepsw_02/.helm/plugins
Creating /home/freepsw_02/.helm/starters
Creating /home/freepsw_02/.helm/cache/archive
Creating /home/freepsw_02/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /home/freepsw_02/.helm.

> helm version
Client: &version.Version{SemVer:"v2.10.0", GitCommit:"9ad53aac42165a5fadc6c87be0dea6b115f93090", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.10.0", GitCommit:"9ad53aac42165a5fadc6c87be0dea6b115f93090", GitTreeState:"clean"}

> kubectl get pods --namespace kube-system
NAME                                                     READY     STATUS    RESTARTS   AGE
tiller-deploy-67d8b477f7-h86fk                           1/1       Running   0          2m

# delete TILLER
> helm reset
```
### Heml 권한 부여

#### 1) kubernetes에 rbac 설정
- helm을 사용하기 위해서는 kubernetes의 tiller에 접근할 수 있어야 한다.
- 하지만, 최신 kubernetes에서는 기본으로 접근을 차단하는 정책을 사용하여, 아래와 같은
- mysql 설치 명령어를 실행하면 해당 helm chart를 찾을 수 없다는 오류가 발생함.
```
> helm repo update
> helm install stable/mysql
```
- helm chart를 찾을 수 없다는 오류 발생시
- RBAC가 활성화 되어 있어서, 해당 권한을 가지지 못할때 발생함.
  - https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/(usecase 2)
- 아래와 같은 명령어를 이용하여 tiller에게 권한을 부여함

- tiller-clusterrolebinding.yaml
```yml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tiller-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: ""
```

```
> kubectl create serviceaccount tiller --namespace kube-system
> kubectl create -f tiller-clusterrolebinding.yaml
> helm init --service-account tiller --upgrade
> helm ls
```

#### 2) RBAC를 비활성화(일단 테스트로 실행할 것이므로..)
```
> kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --user=admin --user=kubelet --group=system:serviceaccounts

> helm ls
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
cold-markhor    1               Mon Aug 27 02:18:19 2018        DEPLOYED        mysql-0.10.1    5.7.14          default  
```

### Heml 사용
```
> kubectl get pods
NAME                                 READY     STATUS    RESTARTS   AGE
cold-markhor-mysql-fc699bdbb-sglms   1/1       Running   0          2m

> helm list
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
cold-markhor    1               Mon Aug 27 02:18:19 2018        DEPLOYED        mysql-0.10.1    5.7.14          default  

> helm status cold-markhor
LAST DEPLOYED: Mon Aug 27 02:18:19 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                                READY  STATUS   RESTARTS  AGE
cold-markhor-mysql-fc699bdbb-sglms  1/1    Running  0         5m

==> v1/Secret
NAME                TYPE    DATA  AGE
cold-markhor-mysql  Opaque  2     5m

==> v1/ConfigMap
NAME                     DATA  AGE
cold-markhor-mysql-test  1     5m

==> v1/PersistentVolumeClaim
NAME                STATUS  VOLUME                                    CAPACITY  ACCESS MODES  STORAGECLASS  AGE
cold-markhor-mysql  Bound   pvc-7769949c-a99f-11e8-9230-42010a920fcd  9G        RWO           standard      5m

==> v1/Service
NAME                TYPE       CLUSTER-IP   EXTERNAL-IP  PORT(S)   AGE
cold-markhor-mysql  ClusterIP  10.7.251.52  <none>       3306/TCP  5m

==> v1beta1/Deployment
NAME                DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
cold-markhor-mysql  1        1        1           1          5m
OTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
cold-markhor-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default cold-markhor-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h cold-markhor-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/cold-markhor-mysql 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}

> helm search mysql
> helm inspect stable/xray
> helm status cold-markhor
> helm inspect values stable/mariadb
> echo '{mariadbUser: user0, mariadbDatabase: user0db}' > config.yaml
> helm install -f config.yaml stable/mariadb
> helm get values  killjoy-lion
> helm history killjoy-lion
> helm list --all
> helm repo list
```

#### [참고] Kubernetes의 RBAC
- https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/
- 쿠버네티스 1.6 이상에서는 RBAC가 기본으로 활성화 됨.
- RBAC는 클러스터의 올바른 관리를 위하여, 사용자/그룹별로 역할을 정의함.
  - operation 권한 부여 (예를 들면, accessing secrets)
  - cluster에 접근할 권한 부여
  - 특정 namespace의 자원 사용 제한 (pods, persistent volumes, deployments)
  - 권한이 부여된 namespace에서만 사용자가 리소스를 볼수 있도록 제한

##### 1). RBAC API Objects
- 쿠버네티스는 모든 리소스(pod, nodes 등)가 모델링된 API Object이며, 각 리소스는 API를 통해서 CRUD 작업을 허용한다.
- 리소스
```
Pods.
PersistentVolumes.
ConfigMaps.
Deployments.
Nodes.
Secrets.
Namespaces.
```

- 리소스에 사용 가능한 operation
```
create
get
delete
list
update
edit
watch
exec
```
- 상위 관점에서 보면, 리소스는 API Group과 관련이 있다.
- 예를 들어, pod는 core API group에 속하고, deployments는 apps API group에 속한다.

##### 2) RBAC의 구성
- kubernetes에서 operation과 resource를 분리하여 RBAC를 관리하기 위해서는 아래의 구성요고가 필요
  - Rules : API group에 속한 자원에서 수행할 수 있는 operation
  - Roles and ClusterRoles : Rule의 집합으로 구성, 클러스터의 API 리소스와 매핑됨
    - Roles : 하나의 namespace에만 적용
    - ClusterRole : 전역 클러스터에 적용 가능

  - Subjects : 클러스터에서 작업을 수해알 객체(entity), 3개 typ으로 구성
    - User Account : 클러스터 외부의 사용자/프로세스를 의미함. 리소스 API object와 관련이 없음
    - Service Accounts : pod 내부에서 실행되는 클러스터의 프로세스를 의미하며, API에 대한 인증을 요구함.
    - Groups : 다수의 account의 그룹.  
  - Rolebindings & ClusterRoleBindings : subject와 role을 연결한다.

##### 3) examples
- http://anti1346.egloos.com/v/4693625 참고
- kubernetes 공식 매뉴얼은 "https://kubernetes.io/docs/concepts/cluster-administration/certificates/"
```

> kubectl create namespace office

# 1. 개인키 생성 (암호화 하지 않음)
> openssl genrsa -out employee.key 2048

# 2. CSR 생성  (인증서 서명 요청을 위해 필요)
> openssl req -new -key employee.key -out employee.csr -subj "/CN=employee/O=bitnami"

# 자체 서명 인증서 생성하기
# kubernetes cluster의 crt/key를 이용하여 하위 인증서를 만든다 (CA, CAkey)
# kubernetes의 CA(ca.key, ca.crt)를 미리 생성해야하는데, 이 부분은 서버의 작업이 별도로 필요함
> openssl x509 -req -in employee.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out employee.crt -days 500
>  mkdir ~/.certs
>  mv employee.* ~/.certs/

> kubectl config set-credentials employee --client-certificate=/home/freepsw_02/.certs/employee.crt  --client-key=/home/freepsw_02/.certs/employee.key
> kubectl config set-context employee-context --cluster=kuar-cluster --namespace=office --user=employee
```


## 3 Chart 의 구성
- wordpress라는 차트 디렉토리 구성
```
wordpress/
  Chart.yaml          # A YAML file containing information about the chart
  LICENSE             # OPTIONAL: A plain text file containing the license for the chart
  README.md           # OPTIONAL: A human-readable README file
  requirements.yaml   # OPTIONAL: A YAML file listing dependencies for the chart
  values.yaml         # The default configuration values for this chart
  charts/             # A directory containing any charts upon which this chart depends.
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

- Chart.yaml  
```yaml
apiVersion: The chart API version, always "v1" (required)
name: The name of the chart (required)
version: A SemVer 2 version (required) - Given a version number MAJOR.MINOR.PATCH, increment the:
kubeVersion: A SemVer range of compatible Kubernetes versions (optional)
description: A single-sentence description of this project (optional)
keywords:
  - A list of keywords about this project (optional)
home: The URL of this project's home page (optional)
sources:
  - A list of URLs to source code for this project (optional)
maintainers: # (optional)
  - name: The maintainer's name (required for each maintainer)
    email: The maintainer's email (optional for each maintainer)
    url: A URL for the maintainer (optional for each maintainer)
engine: gotpl # The name of the template engine (optional, defaults to gotpl)
icon: A URL to an SVG or PNG image to be used as an icon (optional).
appVersion: The version of the app that this contains (optional). This needn't be SemVer.
deprecated: Whether this chart is deprecated (optional, boolean)
tillerVersion: The version of Tiller that this chart requires. This should be expressed as a SemVer range: ">2.0.0" (optional)
```
- version : helm chart에 대한 버전
- appVersion : helm char에 포함된 app(nginx, mysql 등)에 대한 버전



## 4. Developing Templates
- https://docs.helm.sh/chart_template_guide/#the-chart-template-developer-s-guide

### 4-1 Getting Started with a Chart Template
- 먼저 간단한 chart를 생서해 보자
```
> helm create mychart
Creating mychart
```

#### template 구성
- mychart/templates/
- 아래와 같이 기본으로 파일들이 생성된다.
```
NOTES.txt: The “help text” for your chart. This will be displayed to your users when they run helm install.
deployment.yaml: A basic manifest for creating a Kubernetes deployment
service.yaml: A basic manifest for creating a service endpoint for your deployment
_helpers.tpl: A place to put template helpers that you can re-use throughout the chart
```
- 실습을 위해서 기존 파일들은 모두 삭제하자.
```
> rm -rf mychart/templates/*.*
```

##### 첫번째 template 생성하기
- mychart/templates/configmap.yaml
- configMap을 kubernetes에 등록하는 간단한 template
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

- helm 실행하기
```
> helm install ./mychart
NAME: full-coral # --> release name
LAST DEPLOYED: Tue Nov  1 17:36:01 2016
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                DATA      AGE
mychart-configmap   1         1m
```

- 생성된 helm chart의 상세정보 조회하기 (실제 yaml 파일의 내용도 확인 가능, ---로 시작하는 문자열)
```
> helm get manifest full-coral

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

##### 간단한 template 호출해 보기
- {{ }} 기호를 이용하여 template object를 호출한다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

- 작성된 결과를 확인하기 위해서 helm install로 설치하지 않고,
- debug mode를 이용하여 확인가능
```
> helm install --debug --dry-run ./mychart
```

### 4-2 Template에서 사용하는 built-in object
####  Release
- one of the top-level objects that you can access in your templates.

```
Release: This object describes the release itself. It has several objects inside of it:
Release.Name: The release name
Release.Time: The time of the release
Release.Namespace: The namespace to be released into (if the manifest doesn’t override)
Release.Service: The name of the releasing service (always Tiller).
Release.Revision: The revision number of this release. It begins at 1 and is incremented for each helm upgrade.
Release.IsUpgrade: This is set to true if the current operation is an upgrade or rollback.
Release.IsInstall: This is set to true if the current operation is an install.
```
#### Values
- Values passed into the template from the values.yaml file and from user-supplied files. By default, Values is empty.

#### Chart
- The contents of the Chart.yaml file. Any data in Chart.yaml will be accessible here. For example {{.Chart.Name}}-{{.Chart.Version}} will print out the mychart-0.1.0

#### Files
- This provides access to all non-special files in a chart. While you cannot use it to access templates, you can use it to access other files in the chart. See the section Accessing Files for more.
```
Files.Get is a function for getting a file by name (.Files.Get config.ini)
Files.GetBytes is a function for getting the contents of a file as an array of bytes instead of as a string. This is useful for things like images.
```

#### Capabilities
- This provides information about what capabilities the Kubernetes cluster supports.
```
Capabilities.APIVersions is a set of versions.
Capabilities.APIVersions.Has $version indicates whether a version (batch/v1) is enabled on the cluster.
Capabilities.KubeVersion provides a way to look up the Kubernetes version. It has the following values: Major, Minor, GitVersion, GitCommit, GitTreeState, BuildDate, GoVersion, Compiler, and Platform.
Capabilities.TillerVersion provides a way to look up the Tiller version. It has the following values: SemVer, GitCommit, and GitTreeState.
```

#### Template
- Contains information about the current template that is being executed
```
Name: A namespaced filepath to the current template (e.g. mychart/templates/mytemplate.yaml)
BasePath: The namespaced path to the templates directory of the current chart (e.g. mychart/templates).
```


### 4-3 Values Files
- chart 에서 사용할 수 있는 value를 정의하는 파일이다.
- 별도로 생성된 value 파일은 helm install -f <myvalues file> ./mtchart 로 전달 가능
- value 파일을 사용하지 않고, 별도의 value를 전달하는 경우, --set key=value로 전달

#### Run mychart chart examples  
- 아래 예시를 통해서 values.yaml이 어떻게 사용되는지 확인해 보자.
- Charts.YAML
```YAML
apiVersion: v1
appVersion: "1.0"
description: A Helm chart for Kubernetes
name: mychart
version: 0.1.0
```

- values.YAML
```yaml
favorite:
  drink: coffee
  food: pizza
```

- templates/configmap.yaml
```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food  }}
```

- Run chart with debug mode
  - 아래 drink와 food에 values.yaml에 정의된 값이 출력된다.
  - 또한 --set으로 넘긴 favoriteDrink는 기존 값을 override하여 춮력됨(기존 값 무시)
```
> helm install --dry-run --debug --set favoriteDrink=slurm ./mychart

[debug] Created tunnel using local port: '43234'

[debug] SERVER: "127.0.0.1:43234"

[debug] Original chart version: ""
[debug] CHART PATH: /home/freepsw_02/test/mychart

NAME:   inclined-beetle
REVISION: 1
RELEASED: Wed Aug 29 00:05:41 2018
CHART: mychart-0.1.0
USER-SUPPLIED VALUES:
favoriteDrink: slurm

COMPUTED VALUES:
favorite:
  myvalue: "Hello World"
  drink: slurm
  food: pizza
```



### 4-4 Template function and pipelines
- https://docs.helm.sh/chart_template_guide/#pipelines 참고
- Template 객체에 함수를 적용하거나, 함수간의 pipeline 연결을 통하여 값을 전달할 수 있다.
- 아래 예시를 보면, drink값이 없을 경우 "tea"를 사용하며, 해당 값에 ""를 추가하도록 한다.
```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```
```
> helm install --dry-run --debug ./mychart
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
```

### 4-5 Flow control
- https://docs.helm.sh/chart_template_guide/#if-else 참고
- template를 실행하는 조건을 설정하여,
- 특정 상태에 따라서 실행되도록 정의할 수 있다.
```
if/else for creating conditional blocks
with to specify a scope
range, which provides a “for each”-style loop

define declares a new named template inside of your template
template imports a named template
block declares a special kind of fillable template area
```

#### examples
- values에 값이 없을 경우를 확인하고,
- 특정 values를 추가한다.
- values르 추가할때 공백이 발생하는 문제를 해결하기 위해서
- "-"라는 기호를 사용한다. -는 이후의 모든 공백을 제거하는 역할을 함
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee"}}
  mug: true
  {{- end}}
```

#### with를 이용한 조건설정
- https://docs.helm.sh/chart_template_guide/#modifying-scope-using-with
- 아래 예시에서는 value파일의 favorite 객체 하위의 객체를 접근할 수 있도록 범위를 설정했다.
- 만약 with 구문 내부에서 상위의 객체 "Release" 등에 접속하려고 하면 에러가 발생할 것이다.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
```

#### range를 이용한 loop
- 특정 객채에 포함된 하위객체를 순서대로 접근하는 방법
- values.yaml에 아래와 같은 객체를 정의하고, 이를 순서대로 처리하는 경우.
```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

- range 함수를 사용하여 pizzaTopping을 순서대로 출력해준다.
```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}

```

### 4-6 Variables
- template 내부에서 변수를 통해 값을 저장하고, 사용할 수 있다.
- 변수에 값을 할당하는 기호는 := 를 이용한다.
- 일반적으로 $로 선언되는 변수는 선언된 범위(위치) 이하에서만 인식 가능하다.
- 즉 하위블록에서 선언된 변수를 상위에서 사용할 수 없다.
- 아래 예제에서는 $relname이라는 변수에 .Release.Name 값을 할당하고,
- 이를 with 범위내에서 사용하고 있다.
```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- $relname := .Release.Name -}}
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $relname }}
  {{- end }}
```

- range 내부에서 변수를 할당 및 사용하는 방법
```yaml
toppings: |-
    {{- range $index, $topping := .Values.pizzaToppings }}
      {{ $index }}: {{ $topping }}
    {{- end }}
```
- 출력결과
```
toppings: |-
    0: mushrooms
    1: cheese
    2: peppers
    3: onions
```


- values.yaml의 key와 value를 변수로 할당
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end}}
```

### 4-7 Named Template (subTemplate)
- 하나의 template 파일에서 여러개의 template를 정의하고 사용할 수 있다.
- 기억할 내용으로는 template명은 전역으로 사용이 가능하다. (하나의 template에 여러개 template가 있더라도..)
- 따라서 template 명을 만들때는 chart명을 이용하여 구분이 가능하도록 설정하는 것을 권장한다.
- {{ define "mychart.labels" }}

#### template 파일 구성
- template폴더 아래의 파일은 kubernetes manifest를 정의한다. (NOTES.txt 제외)
- 특이사항으로 _ 로  시작하는 파일은 manifest를 정의하지 않고, partials과 helper애 대한 정보만 유지

#### Define을 이용한 template 생성
```YAML
{{ define "MY.NAME" }}
  # body of template here
{{ end }}
```

- 아래와 같이 template block을 생성하여 재사용 가능
- mychart.labels 템플릿을 사용,
```yaml
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

- 일반적으로 이렇게 define 된 template는 _ helpers.tpl 에 저장하고,
- 여기에 정의된 template을 yaml 파일에서 사용할 수 있다.
- template 명은 전역으로 사용할 수 있으므로,
- tpl에 정의된 템플릿은 어디서나 사용 가능하다.

```tpl (_helpers.tpl)
{{/* Generate basic labels */}}
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

- configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

#### 템플릿의 적용범위 설정
- 생성된 템플릿이 사용할 수 있는 object의 범위를 사전에 설정할 수 있다. .
- "." 을 지정하면, 최상위 객체를 모두 사용할 수 있다는 의미가 되며,
- ".Chart" 와 같이 특정범위를 지정할 수 도 있다.
- 만약 범위를 지정하지 않고 사용하는 경우, 상위 객체를 인식하지 못하는 오류가 발생할 수 있음
```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" . }}
```

#### Include 함수
- 기존 방식의 template은 들여쓰기가 되어있는 그대로 출력을 하게 된다.
- 따라서 상황에 따라서 다른 들여쓰기가 필요한 경우에는 활용하기 어렵다.
- 아래의 예시를 보자.
``` tlc file
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}+{{ .Release.Time.Seconds }}"
{{- end -}}
```
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{ template "mychart.app" .}}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ template "mychart.app" . }}
```
- 위 코드를 실행하면 원하는 형식으로 화면이 출력되지 않는다.
- 아래와 같이 들여쓰기가 밀려버리는 현상이 발생한다.
```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: measly-whippet-configmap
  labels:
    app_name: mychart
app_version: "0.1.0+1478129847"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
app_version: "0.1.0+1478129847"
```
- 이는 템플릿은 함수가 아니라 실행하는 액션이이 때문에,
- 단지 화면에 해당 라인을 출력할 뿐이다. (오른쪽 정렬에 맞추어서)
- 그래서 include 함수를 이용하여 들여쓰기를 지원하도록 한다.
- 각 include 함수에서 pipeline을 통해 indent를 추가할 수 있다.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{ include "mychart.app" . | indent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ include "mychart.app" . | indent 2 }}
```

- 위 코드를 실행하면 정상적으로 들여쓰기가 적용되어 출력된다.
```YAML
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-mole-configmap
  labels:
    app_name: mychart
    app_version: "0.1.0+1478129987"
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
  app_name: mychart
  app_version: "0.1.0+1478129987"
```

### 4-8 Accessing Files Inside Templates
- helm은 .Files 객체를 이용해서 파일에 접근할 수 있도록 지원
- 별도 파일을 추가할 수는 있지만, chart는 1M를 넘지 않아야 한다. (스토리지 이슈로 인하여)
- 또한 특정 파일은 접근이 금지된다
  - templates 아래의 파일들
  - .helmignore에 지정된 파일들
- 아래 예시를 통해 살펴보자
#### Basic example
- 3개의 파일을 생성하고, 템플릿에서 3개 파일을 읽어오는 예시
```
> cd mychart
> echo "message = Hello from config 1" >> config1.toml
> echo "message = Hello from config 2" >> config2.toml
> echo "message = Hello from config 3" >> config3.toml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  {{- $files := .Files }}
  {{- range tuple "config1.toml" "config2.toml" "config3.toml" }}
  {{ . }}: |-
    {{ $files.Get . }}
  {{- end }}
```

```
> helm install --debug --dry-run ./mychart

# Source: mychart/templates/file.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mortal-dachshund-configmap
data:
  config1.toml: |-
    message = Hello from config 1

  config2.toml: |-
    message = Hello from config 2

  config3.toml: |-
    message = Hello from config 3
```

### 4-9 Subchart
- chart는 여러개의 하위 chart를 가질 수 있으며,
- 각 subchart는 각각의 value와 template 파일들을 가지고 있다.
- subchart에 대한 중요한 개념들
  - 1. subchart는 독립적인 하나의 chart이다. 상위 chart에 종속되지 않는다.
  - 2. 1번의 이유로 subchart는 상위차트의 갑에 접근할 수 없다.
  - 3. 상위 차트는 subchart의 값을 override할 수 있다.
  - 4. helm은 모든 chart들이 접근할 수 있는 global value의 개념을 제공한다.

#### subchart 생성 및 테스트
```
> cd mychart/charts/
> helm create mysubchart
> rm -rf mysubchart/templates/*.*
```

- values.yaml 파일에 값 설정
```YAML
dessert: cake
```

- 실행할 template 파일 생성
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
```

- subchart 실행
```
> helm install --dry-run --debug mychart/charts/mysubchart

---
# Source: mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elder-elephant-cfgmap2
data:
  dessert: cake
```

#### subchart에서 활용할 수 있도록 상위 chart의 value를 지정해 보자
- mychart의 values.yaml을 아래와 같이 변경한다.
- 맨 아래 두번째 줄을 보면, subchart의 이름을 지정하고, 값을 정의했다.
- 이는 해당 값들은 지정된 subchart에서 접근할 수 있다.
```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream
```

- 이번엔 상위 차트인 mychart를 실행해 보자.
- 아까와 다르게 mysubchart에서 ice cream 이라는 값이 출력되는 것을 볼 수 있다.
```
> helm install --dry-run --debug mychart

# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kissed-chinchilla-cfgmap2
data:
  dessert: ice cream
```


#### Global chart value
- 위의 예시에서 subchart의 값을 사용하기 위해서는
- subchart명을 명시하고, 그 하위에 정의된 값만 사용이 가능했다.
- global value는 그런 제약없이 global로 선언된 모든 변수를 subchart에서 사용가능하다.


```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream

global:
  salad: caesar
```

- 아래와 같이 global 변수를 활용.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
  salad: {{ .Values.global.salad }}
```

- 다시 chart를 실행해 보면 subchart에서 global value에 접근이 가능하다.
- 실행은 반드시 상위 chart에서 실행해야 subchar에서 global 값에 접근이 가능하다.
```
> helm install --dry-run --debug mychart

# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: silly-snake-cfgmap2
data:
  dessert: ice cream
  salad: caesar
```
