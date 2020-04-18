# 1. Create AWS Kubernetes Cluster using kops
- https://blog.naver.com/freepsw/221758787689 참고 
```shell
kops create cluster --name=k8s.freepsw.com \
  --state=s3://kops-state-freepsw --zones=ap-northeast-2a \
  --node-count=2 --node-size=t2.micro --master-size=t2.micro \
  --dns-zone=k8s.freepsw.com


kops update cluster k8s.freepsw.com --yes \
    --state=s3://kops-state-freepsw

kops validate cluster --state=s3://kops-state-freepsw

kops delete cluster --name k8s.freepsw.com  --yes --state=s3://kops-state-freepsw

```
