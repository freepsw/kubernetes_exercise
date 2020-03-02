# Udemy The Complete Kubernetesrnetes Guide 실습 코드 정리

## Section 1. Course Introduction

## Section 2. Introduction to Kubernetes


## Section 3. Basics

## Section 4. Advanced Topics



## ETC 1. Google GKE 사용을 위한 설정 및 GKE 생성
### 1. Install gcloud sdk on mac
- https://cloud.google.com/sdk/docs/quickstart-macos

#### Install gcloud
```shell
> cd /Users/skiper
> wget https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-256.0.0-darwin-x86_64.tar.gz

> tar xvf google-cloud-sdk-256.0.0-darwin-x86_64.tar.gz

# gcloud 설치
> ./google-cloud-sdk/install.sh
......
Modify profile to update your $PATH and enable shell command
completion?

Do you want to continue (Y/n)?  Y  # gcloud 명령어 path를 추가할지 여부 (Y선택)

The Google Cloud SDK installer will now prompt you to update an rc
file to bring the Google Cloud CLIs into your environment.

# (gcloud PATH를 어떤 shell 설정에 추가할지 경로 지정, 나는 Mac을 사용하므로 기본 shell인 zsh를 선택)
# 이렇게 선택하고 엔터를 클릭하면, 자동으로 gcloud path가 shell 환경에 적용이 된다.
Enter a path to an rc file to update, or leave blank to use
[/Users/skiper/.zshrc]:  /Users/skiper/.zshrc  
No changes necessary for [/Users/skiper/.zshrc].

For more information on how to get started, please visit:
  https://cloud.google.com/sdk/docs/quickstarts

> which gcloud
/Users/skiper/work/DevTools/google-cloud-sdk/bin/gcloud

```

#### Initialze gcloud sdk environment
```shell
> gcloud init


# gcloud로 GGP에 연결하기 위해서는 초기 Browser에서 인증을 해야함.
# 아래 링크로 접속하여, GGP 계정으로 로그인한다.
You must log in to continue. Would you like to log in (Y/n)?  Y

Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/auth?redirect_uri=http%3A%2F%2Flocalhost%3A8085%2F&prompt=select_account&response_type=code&client_id=32555940559.apps.googleusercontent.com&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fappengine.admin+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcompute+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Faccounts.reauth&access_type=offline

Updates are available for some Cloud SDK components.  To install them,
please run:
  $ gcloud components update

You are logged in as: [freepsw@xxxxxxx.com].

# gcloud로 작업할 프로젝트를 선택한다.
Pick cloud project to use:
 [1] test-my-platform
 [2] key-prism-xxxxxx
 [3] Create a new project
Please enter numeric choice or text value (must exactly match list
item):  1

Your current project has been set to: [ds-ai-platform].

Do you want to configure a default Compute Region and Zone? (Y/n)?  n

Your Google Cloud SDK is configured and ready to use

Run gcloud help config to learn how to change individual settings


# gcloud 테스트
> gcloud auth list
           Credentialed Accounts
ACTIVE             ACCOUNT
                  freepsw@xxxxxx.com
```




### 3. Create Google GKE Cluster
- https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster
#### Enable "Kubernetes Engine API"
- GKE를 설치하기 위해서는 google cloud api service를 활용할 수 있는 권한이 필요함.
- 아래 서비스 목록을 보면 "Kugernetes Engine API" 서비스 권한이 필요함
- 현재 서비스 목록을 보면, 해당 서비스가 보이지 않음.
```shell
# 서비스 목록 조회
> gcloud services list
NAME                              TITLE
bigquery.googleapis.com           BigQuery API
bigquerystorage.googleapis.com    BigQuery Storage API
cloudapis.googleapis.com          Google Cloud APIs
cloudbuild.googleapis.com         Cloud Build API
clouddebugger.googleapis.com      Stackdriver Debugger API
cloudprofiler.googleapis.com      Stackdriver Profiler API
cloudtrace.googleapis.com         Stackdriver Trace API
compute.googleapis.com            Compute Engine API
containerregistry.googleapis.com  Container Registry API
datastore.googleapis.com          Cloud Datastore API
logging.googleapis.com            Stackdriver Logging API
ml.googleapis.com                 AI Platform Training & Prediction API
monitoring.googleapis.com         Stackdriver Monitoring API
oslogin.googleapis.com            Cloud OS Login API
pubsub.googleapis.com             Cloud Pub/Sub API
servicemanagement.googleapis.com  Service Management API
serviceusage.googleapis.com       Service Usage API
sql-component.googleapis.com      Cloud SQL
stackdriver.googleapis.com        Stackdriver API
storage-api.googleapis.com        Google Cloud Storage JSON API
storage-component.googleapis.com  Cloud Storage
```

##### "Kubernetes Engine API" 사용설정
- Google Cloud Console에서 Kubernetes Engine 페이지로 이동합니다.
- 프로젝트를 만들거나 선택합니다.
- API 및 관련 서비스가 사용 설정될 때까지 기다립니다. 몇 분 정도 걸릴 수 있습니다.
- Google Cloud Platform 프로젝트에 결제가 사용 설정되어 있는지 확인합니다. 프로젝트에 결제가 사용 설정되어 있는지 확인하는 방법을 알아보세요.

#### install kubectl components
```shell
> gcloud components install kubectl
```

#### gcloud config 설정
```shell
> gcloud config set project ds-ai-xxx # project id 입력
Updated property [core/project].
> gcloud config set compute/region us-west1
Updated property [compute/region].
```


#### Create Cluster
```shell
> gcloud container clusters create freepsw-cluster --zone us-central1-a --cluster-version 1.15
# 인증 설정 (생성한 GKE와 통신 연결)
> gcloud container clusters get-credentials freepsw-cluster --zone us-central1-a
Fetching cluster endpoint and auth data.
kubeconfig entry generated for freepsw-cluster.


# deployment  생성 및 조회
> kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
deployment.apps/hello-server created
> kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-57c495f96c-pql9c   1/1     Running   0          13s
```

#### Delete Cluster
```shell
> gcloud container clusters delete freepsw-cluster --zone us-central1-a
>
```
