## Section 7: Continuous Development with Kubernetes

### Introduction to Skaffold
- google의 오픈소스로 작성된 application을 빌드/push/deploy 하는 workflow를 다루는 오픈소스  
- https://skaffold.dev/
- https://github.com/GoogleContainerTools/skaffold
  - build : docker build 수행
  - pushing : docker repository 에 등록
  - deploy : k8s cluster에 배포
- 예를 들어, application에 변경이 발생하면 build/push/deploy 과정을 통해서 k8s에 빠르게 배포한다.
- 기존 CI/CD Pipeline과 통합하여 활용 가능
- plugins
  - Build : docker(local 환경)), kaniko(local + k8s 환경), cloud  
    - https://github.com/GoogleContainerTools/kaniko
  - Deploying : kubectl과 helm을 이용하여 간단하게 실행 가능


  ﻿
