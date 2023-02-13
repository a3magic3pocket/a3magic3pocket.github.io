---
title: Kubernetes(minikube)를 이용하여 웹 애플리케이션 운영하기
date: 2023-02-14 00:42:00 +0900
categories: [kubernetes]
tags: [kubernetes, minikube, docker, git, git-action, dockerhub, cicd] # TAG names should always be lowercase
---
# Kubernetes란
- [쿠버네티스란 무엇인가?](https://kubernetes.io/ko/docs/concepts/overview/){:target="_blank"}
- 컨테이너 오케스트레이션 툴이다.
- 쿠버네티스는 [Docker swarm mode](https://a3magic3pocket.github.io/posts/docker-swarm-mode-deploy-application/){:target="_blank"}와 비슷하나  
대규모 애플리케이션에 보다 더 적합하다.
    - [Docker Swarm vs Kubernetes: how to choose a container orchestration tool](https://circleci.com/blog/docker-swarm-vs-kubernetes/){:target="_blank"}
- minikube란 쿠버네티스에 익숙해질 수 있도록 제공하는 실험용 툴이다.
    - [hello-minikube](https://kubernetes.io/ko/docs/tutorials/hello-minikube/){:target="_blank"}
- 본격적으로 쿠버네티스를 사용할 경우,  
    3개의 노드가 필수적으로 필요하기에  
    실험은 minikube로 진행한다.


## 실험환경
- 목표
    - [simple-locker](https://coinlocker.link/){:target="_blank"}를 minikube를 사용하여 배포해보고  
        CI / CD 파이프라인에 연동해본다.
- 가정
    - 단일 노드에서 배포하는 것을 가정한다.
    - 노드 os는 ubuntu 22.04 이다.
    - 배포하고자 하는 웹 애플리케이션은 [simple-locker](https://www.coinlocker.link/){:target="_blank"}이다.
- simple-locker의 구조
    - <a href="/assets/img/2023-02-14-kubernetes-deploy-application/00-simple-locker-structure.jpg" target="_blank"><img src="/assets/img/2023-02-14-kubernetes-deploy-application/00-simple-locker-structure.jpg" width="100%"></a>
- kubernetes 매니페스트 파일
    - [a3magic3pocket/simple-manifest: simple-manifest (github.com)](https://github.com/a3magic3pocket/simple-manifest){:target="_blank"}
- 매니페스트 파일
    - 매니페스트란
        - 설정들이 기록된 yaml 파일들을 말한다.
        - 쿠버네티스에서 yaml 파일을 보고 환경을 동기화시킨다.
    - 사용하는 매니페스트 목록
        - config.yml
        - secret.yml
        - sqlite3-pv.yml
        - sqlite3-pvc.yml
        - webhook-endpoints.yml
        - webhook-svc.yml
        - cluster-issuer.yml
        - simple-api.yml
        - simple-web.yml
        - ingress.yml


## Manifest: config.yml
- 설명
    - [configMap](https://kubernetes.io/ko/docs/concepts/configuration/configmap/){:target="_blank"}
    - 모든 서비스에서 참조할 수 있는 컨피그맵이다.
- config.yml  
    ```yml
    apiVersion: v1
    kind: ConfigMap
    metadata:
    name: simple-api
    data:
    useK8s: "yes"
    frontendUrlList: "https://your-frontend-url.com,https://www.your-frontend-url.com"
    ginMode: "release"
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
    name: simple-web
    data:
    backendOrigin: "your-backend-url.com"
    frontendOrigin: "your-frontend-url.com"
    ```


## Manifest: secret.yml
- 설명
    - [Secret](https://kubernetes.io/ko/docs/concepts/configuration/secret/){:target="_blank"}
    - 모든 서비스에서 참조할 수 있는 시크릿이다.
    - 컨피그와 다른 점은 암호화되어 저장된다.
    - 개별 시크릿 크기는 1MiB로 제한된다.
    - 시크릿 내용은 base64로 인코딩되어 있어야 한다.  
        ex) echo "my-secret" | base64
- secret.yml  
    ```yml
    apiVersion: v1
    kind: Secret
    metadata:
    name: simple-api-secret
    type: Opaque
    data:
    identityKey: your-identityKey
    authSecretKey: your-authSecretKey
    ```


## Manifest: sqlite3-pv.yml
- 설명
    - [persistent-volumes](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/){:target="_blank"}
    - pod는 stateless하기 때문에 pod의 저장소에 sqlite3를 저장하여   
        운영할 수 없다.
    - 그래서 PV(persistance volume)을 할당하고 이를 서비스 배포하여  
        다른 서비스에서 이 볼륨을 참조할 수 있도록 한다.
    - 여기서 volume이 hostPath를 갖는데  
        이는 호스트노드의 일정 경로에 PV을 저장하겠다는 의미이다.  
        [hostpath](https://kubernetes.io/ko/docs/concepts/storage/volumes/#hostpath){:target="_blank"}
    - 실험 환경이 단일 노드이기에 가능한 방법이며  
        실제 production 환경에서는 nfs 등의 볼륨으로 바꿔야할 것이다.  
        [nfs](https://kubernetes.io/ko/docs/concepts/storage/volumes/#nfs){:target="_blank"}
- sqlite3-pv  
    ```yml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
    name: sqlite3-pv-volume
    labels:
        type: local
    spec:
    storageClassName: manual
    capacity:
        storage: 500Mi
    accessModes:
        - ReadWriteMany
    hostPath:
        path: "/mnt/data"
    ```


## Manifest: sqlite3-pvc.yml
- 설명
    - 참고
        - [볼륨과 클레임 라이프사이클](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/){:target="_blank"}
        - [[k8s] Volume - PV/PVC(퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임)](https://kimjingo.tistory.com/153){:target="_blank"}
    - 정적 프로비저닝
        - PV는 리소스이고  
            PVC(persistent volume claim)는 리소스에 대한 요청을 제어한다.
        - pod는 PVC에 요청을 하고 PVC는 문제가 없는 경우 PV로 요청을 전달한다.
        - PVC요청보다 pv 스펙이 부족한 경우 pod의 요청이 실패
        - PV는 하나의 PVC와 연결할 수 있다.
        - PV와 PVC는 storageClass로 연결된다.
- sqlite3-pvc.yml  
    ```yml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: sqlite3-pv-claim
    spec:
    storageClassName: manual
    accessModes:
        - ReadWriteMany
    resources:
        requests:
        storage: 500Mi
    ```


## Manifest: webhook-endpoints.yml
- 설명
    - [Endpoints API](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/endpoints-v1/){:target="_blank"}
    - [endpoints](https://kubernetes.io/docs/concepts/services-networking/service/#endpoints){:target="_blank"}
    - 엔드포인트를 설정하여  
        서비스에서 해당 엔드포인트를 참조할 수 있게 한다.
- webhook-endpoints.yml  
    ```yml
    apiVersion: v1
    kind: Endpoints
    metadata:
    name: webhook-svc
    subsets:
        - addresses:
            - ip: your-webhook-ip
            ports:
            - port: 9000
    ```


## Manifest: webhook-svc.yml
- 설명
    - webhook-svc 엔드포인트와 연결되는 서비스이다.
    - 서비스가 pod와 연결될때  
        실제로는 중간에 엔드포인트를 통한다고 한다.  
        [[Kubernetes] Endpoints란](https://yoo11052.tistory.com/193){:target="_blank"}
    - 여기서는 webhook url을 엔드포인트에 직접 등록하므로써  
        각 서비스에서 웹훅 URL로 접근할 수 있게 설정하였다.
- webhook-svc.yml  
    ```yml
    apiVersion: v1
    kind: Service
    metadata:
    name: webhook-svc
    spec:
    ports:
        - protocol: TCP
        port: 80
        targetPort: 9000
    ```


## Manifest: cluster-issuer.yml
- 설명
    - [Kubernetes 에서 Cert-manager로 letsencrypt 인증서 발급/설정](https://cleanupthedesk.tistory.com/43){:target="_blank"}
    - [쿠버네티스 cert-manager로 let's encrypt 인증서 발급](https://malwareanalysis.tistory.com/126){:target="_blank"}
    - [cert manager installation](https://cert-manager.io/docs/installation/){:target="_blank"}
    - certbot 으로 ssl 인증서를 발급할때 사용하는 issuer이다.
    - 이때 생성된 인증서를 ingress에 적용하여 https 접속이 가능하게 한다.
- cluster-issuer.yml  
    ```yml
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
    name: letsencrypt-staging
    spec:
    acme:
        # The ACME server URL
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        # Email address used for ACME registration
        email: your-email@gmail.com
        # Name of a secret used to store the ACME account private key
        privateKeySecretRef:
        name: letsencrypt-staging
        # Enable the HTTP-01 challenge provider
        solvers:
        - http01:
            ingress:
            class:  nginx
    ---
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
    name: letsencrypt-prod
    spec:
    acme:
        # The ACME server URL
        server: https://acme-v02.api.letsencrypt.org/directory
        # Email address used for ACME registration
        email: your-email@gmail.com
        # Name of a secret used to store the ACME account private key
        privateKeySecretRef:
        name: letsencrypt-prod
        # Enable the HTTP-01 challenge provider
        solvers:
        - http01:
            ingress:
            class: nginx
    ```


## Manifest: simple-api.yml
- 설명
    - [deployment](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/){:target="_blank"}
    - simple-locker의 api deployment이다.
- simple-api.yml  
    ```yml
    # deployment 설정
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: simple-api
    spec:
    selector:
        matchLabels:
        app: simple-api
    replicas: 2
    template:
        metadata:
        labels:
            app: simple-api
        spec:
        # 볼륨, pvc에 요청한다.
        volumes:
            - name: sqlite3-pv-storage
            persistentVolumeClaim:
                claimName: sqlite3-pv-claim
        # 컨테이너 관련 설정
        containers:
            - name: simple-api
            image: a3magic3pocket/simple-api:0.0.7
            ports:
                - containerPort: 8080
            volumeMounts:
                - mountPath: "/root/api/sqlite3"
                name: sqlite3-pv-storage
            env:
                - name: USE_K8S
                valueFrom:
                    configMapKeyRef:
                    name: simple-api
                    key: useK8s
                - name: FRONTEND_URL_LIST
                valueFrom:
                    configMapKeyRef:
                    name: simple-api
                    key: frontendUrlList
                - name: GIN_MODE
                valueFrom:
                    configMapKeyRef:
                    name: simple-api
                    key: ginMode
                - name: IDENTITY_KEY
                valueFrom:
                    secretKeyRef:
                    name: simple-api-secret
                    key: identityKey
                - name: AUTH_SECRET_KEY
                valueFrom:
                    secretKeyRef:
                    name: simple-api-secret
                    key: authSecretKey
    ---
    # 서비스 설정
    apiVersion: v1
    kind: Service
    metadata:
    name: simple-api
    labels:
        app: simple-api
    spec:
    ports:
        - port: 8080
        targetPort: 8080
    selector:
        app: simple-api
    # type: LoadBalancer
    # type: NodePort
    type: ClusterIP
    
    ```


## Manifest: simple-web.yml
- 설명
    - [deployment](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/){:target="_blank"}
    - simple-locker의 프론트 서버 deployment이다.
- simple-web.yml  
    ```yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: simple-web
    spec:
    selector:
        matchLabels:
        app: simple-web
    replicas: 2
    template:
        metadata:
        labels:
            app: simple-web
        spec:
        containers:
            - name: simple-web
            image: a3magic3pocket/simple-web:0.0.19
            ports:
                - containerPort: 3000
            env:
                - name: BACKEND_ORIGIN
                valueFrom:
                    configMapKeyRef:
                    name: simple-web
                    key: backendOrigin
                - name: FRONTEND_ORIGIN
                valueFrom:
                    configMapKeyRef:
                    name: simple-web
                    key: frontendOrigin
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: simple-web
    labels:
        app: simple-web
    spec:
    ports:
        - port: 3000
        targetPort: 3000
    selector:
        app: simple-web
    # type: LoadBalancer
    # type: NodePort
    type: ClusterIP
    
    ```


## Manifest: ingress.yml
- 설명
    - [ingress](https://kubernetes.io/ko/docs/concepts/services-networking/ingress/){:target="_blank"}
    - [NGINX 인그레스(Ingress) 컨트롤러로 Minikube에서 인그레스 설정하기](https://kubernetes.io/ko/docs/tasks/access-application-cluster/ingress-minikube/){:target="_blank"}
    - 외부의 모든 요청은 ingress로 먼저 통하게 되고  
        ingress는 적절히 서비스에 해당 요청을 분배한다.
    - 여기서는 nginx ingress controller를 사용한다.  
        [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/){:target="_blank"}
- ingress.yml    
    ```yml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
    name: simple-ingress
    annotations: 
        kubernetes.io/ingress.class: 'nginx' 
        nginx.ingress.kubernetes.io/rewrite-target: / 
        nginx.ingress.kubernetes.io/ssl-passthrough: 'true'
        cert-manager.io/issuer: "letsencrypt-prod"
        # cert-manager.io/issuer: "letsencrypt-staging"
    spec:
    tls:
        - hosts:
            - www.your-frontend-url.com
            - your-frontend-url.com
            - your-backend-url.com
            - your-webhook-url.com
            secretName: simple-tls
    rules:
        - host: "your-backend-url.com"
        http:
            paths:
            - pathType: Prefix
                path: / 
                backend:
                service:
                    name: simple-api
                    port: 
                    number: 8080
        - host: "your-frontend-url.com"
        http:
            paths: 
            - pathType: Prefix
                path: / 
                backend:
                service:
                    name: simple-web 
                    port: 
                    number: 3000
        - host: "www.your-frontend-url.com"
        http:
            paths: 
            - pathType: Prefix
                path: / 
                backend:
                service:
                    name: simple-web 
                    port: 
                    number: 3000
        - host: "your-webhook-url.com"
        http:
            paths:
            - pathType: Prefix
                path: /
                backend:
                service:
                    name: webhook-svc
                    port:
                    number: 9000
    ```


## 실험시작
- minikube 설치, kubectl 설치, minikube 실행  
    ```bash
    # 우분투 환경 구성
    sudo apt-get update && sudo apt-get upgrade
    
    # minikube 설치
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    
    # 도커 설치(기존 도커가 있다면 삭제하고 진행)
    sudo apt-get remove docker docker-engine docker.io containerd runc
    
    ## 필요한 프로그램 설치
    sudo apt-get install \
        ca-certificates \
        curl \
        gnupg \
        lsb-release
    
    ## Add Docker’s official GPG key:
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    
    ## Use the following command to set up the stable repository
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    
    # 설치
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io
    
    # minikube default driver 도커로 변경.
    minikube config set driver docker
    
    # 지금 접속한 유저를 docker group에 포함시킴
    sudo usermod -aG docker $USER && newgrp docker
        
    # kubectl 설치
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    
    # minikube 시작
    minikube start --memory=1937mb
    ```
- 매니페스트 파일 확보  
    ```bash
    git clone https://github.com/a3magic3pocket/simple-manifest.git
    cd simple-manifest
    # 본인 환경에 맞에 매니페스트 파일을 수정합니다.
    # ex) your-frontend-url.com -> realmyapp.com
    ```
- ingress 애드온 설치 및 매니페스트 적용(순서 중요)  
    ```bash
    # 시크릿 및 컨피그 배포
    kubectl apply -f config.yml
    kubectl apply -f secret.yml
    
    # pv 및 pvc 배포
    kubectl apply -f sqlite3-pv.yml
    kubectl apply -f sqlite3-pvc.yml
    
    # webhook url 엔드포인트 서비스 배포
    kubectl apply -f webhook-endpoints.yml
    kubectl apply -f webhook-svc.yml
    
    # 제대로 생성되었나 파드 확인
    # kubectl get pod -o wide
    
    # cerbot 설정
    kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.11.0/cert-manager.yaml
    kubectl apply -f cluster-issuer.yml
    
    # 확인
    kubectl get pods -n cert-manager
    
    # 디플로이먼트 배포
    kubectl apply -f ./simple-api/simple-api.yml
    kubectl apply -f ./simple-web/simple-web.yml
    
    # ingress 관련 addon 활성화
    minikube addons enable ingress
    minikube addons enable ingress-dns
    
    # 인그레스 적용
    kubectl apply -f ingress.yml
    
    # 서비스가 제대로 떴는지 확인 
    curl https://$(minikube ip) -H "Host: [your-domain.com]" -v -k
    ```
- rinetd 설치 및 실행
    - 설명
        - minikube라서 그런지 minikube ip가 외부로 노출되지 않는다.
        - rinetd를 사용해서 서버 80,433포트로 오는  
            모든 TCP 트래픽을 minikube ip로 포워딩한다.
        - [rinetd 사용법](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=alice_k106&logNo=221091150196){:target="_blank"}
    - 실행  
        ```bash
        # rinetd 다운로드 페이지
        # https://github.com/samhocevar/rinetd/releases/tag/v0.73
        # rinetd 다운로드
        wget https://github.com/samhocevar/rinetd/releases/download/v0.73/rinetd-0.73.tar.gz
        
        # 압축 풀기
        tar -xzvf rinetd*.tar.gz
        
        # gcc make를 위해 build essential 받아줌
        sudo apt-get install build-essential
        
        # netstat 설치
        sudo apt-get install net-tools
        
        # 혹시 80 포트가 물려있는지 확인
        netstat -l --numeric-ports -p |grep 80
        
        # minikube ip 확인
        echo $(minikube ip)
        
        # sudo vim /etc/rinetc.conf에 아래 내용 추가
        0.0.0.0 80 [minicube ip] 80
        0.0.0.0 443 [minicube ip] 443
        
        # rinetd 설치
        cd ./rinetd-0.73
        ./bootstrap
        ./configure
        make
        make install
        
        # foreground 실행
        sudo rinetd -c /etc/rinetc.conf -f
        ```
- webhook 설정
    - deploy.sh 스크트립트 외에는 [docker-swarm-mode](https://a3magic3pocket.github.io/posts/docker-swarm-mode-deploy-application/){:target="_blank"}와 동일하다.
    - deploy.sh   
        ```bash
        kubectl apply -f config.yml
        kubectl apply -f secret.yml
        
        kubectl apply -f sqlite3-pv.yml
        kubectl apply -f sqlite3-pvc.yml
        
        kubectl apply -f webhook-endpoints.yml
        kubectl apply -f webhook-svc.yml
        
        kubectl apply -f simple-api.yml
        kubectl apply -f simple-web.yml
        
        kubectl apply -f ingress.yml
        ```
- 자주 쓰는 명령어
    - [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/){:target="_blank"}
    - 명령어  
        ```bash
        # version check
        minikube version
        
        # minikube 실행
        minikube start
        
        
        # 모든 파드 조회
        kubectl get pods -o wide    
        
        # deployments 확인
        kubectl get deployments
        
        # namespace를 포함하여 모든 deployments 조회
        kubectl get deploy -A
        
        # pods 확인
        kubectl get pods -o wide
        
        # nodes 확인 
        kubectl get nodes
        
        # 서비스 확인
        kubectl get services
        
        # 특정 파드 상세 명세 조회
        kubectl describe pods [pod-name]
        
        # namespace 확인
        kubectl get namespace
        
        
        # yaml 파일 적용
        kubectl apply -f [yaml file path]
        
        # ex) deployment 적용
        kubectl apply -f simple-api.yml
        kubectl apply -f simple-web.yml
        
        
        # replicas 수 조정
        kubectl scale --current-replicas=2 --replicas=3 deployment/[deployment-name]
        
        
        # deployment 삭제
        kubectl delete deploy deploymentname -n namespacename
        
        
        # yaml 로그 확인
        kubectl logs -f [yaml file path]
        
        # ex) deployment 로그 확인
        kubectl logs -f deployment/simple-api
        kubectl logs -f deployment/simple-web
        
        
        # minikube 이미지 삭제
        # Reference by : https://stackoverflow.com/questions/68855754/how-can-i-remove-an-image-from-minikube
        minikube image rm [your-image-id]
        ```


## 소감
- minikube 조차 manager 노드를 실행하려면   
    cpu 2core, 2GB memory 사양이 필요하다.  
    그만큼 작은 규모의 애플리케이션에는  
    쿠버네티스가 과하게 무거운 것 같다.  
- 하지만 애플리케이션 규모가 엄청나게 크고  
    각종 배포가 수동으로 중앙에서 제어하기 힘들 정도로 자주 일어나기 시작하면  
    쿠버네티스 구조를 일괄 도입하는게 더 관리비용이 저렴해질 것이라 생각한다.  
- simple-locker도 처음엔 minikube로 올렸지만   
    AWS 프리티어로 올릴 수가 없어  
    현재는 docker-swarm mode로 전환한 상태이다.  


## 참고
- [쿠버네티스란 무엇인가?](https://kubernetes.io/ko/docs/concepts/overview/){:target="_blank"}
- [Docker swarm mode](https://a3magic3pocket.github.io/posts/docker-swarm-mode-deploy-application/){:target="_blank"}
- [Docker Swarm vs Kubernetes: how to choose a container orchestration tool](https://circleci.com/blog/docker-swarm-vs-kubernetes/){:target="_blank"}
- [hello-minikube](https://kubernetes.io/ko/docs/tutorials/hello-minikube/){:target="_blank"}
- [simple-locker](https://coinlocker.link/){:target="_blank"}
- [a3magic3pocket/simple-manifest: simple-manifest (github.com)](https://github.com/a3magic3pocket/simple-manifest){:target="_blank"}
- [configMap](https://kubernetes.io/ko/docs/concepts/configuration/configmap/){:target="_blank"}
- [Secret](https://kubernetes.io/ko/docs/concepts/configuration/secret/){:target="_blank"}
- [persistent-volumes](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/){:target="_blank"}
- [hostpath](https://kubernetes.io/ko/docs/concepts/storage/volumes/#hostpath){:target="_blank"}
- [nfs](https://kubernetes.io/ko/docs/concepts/storage/volumes/#nfs){:target="_blank"}
- [볼륨과 클레임 라이프사이클](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/){:target="_blank"}
- [[k8s] Volume - PV/PVC(퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임)](https://kimjingo.tistory.com/153){:target="_blank"}
- [Endpoints API](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/endpoints-v1/){:target="_blank"}
- [endpoints](https://kubernetes.io/docs/concepts/services-networking/service/#endpoints){:target="_blank"}
- [[Kubernetes] Endpoints란](https://yoo11052.tistory.com/193){:target="_blank"}
- [Kubernetes 에서 Cert-manager로 letsencrypt 인증서 발급/설정](https://cleanupthedesk.tistory.com/43){:target="_blank"}
- [쿠버네티스 cert-manager로 lets encrypt 인증서 발급](https://malwareanalysis.tistory.com/126){:target="_blank"}
- [cert manager installation](https://cert-manager.io/docs/installation/){:target="_blank"}
- [deployment](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/){:target="_blank"}
- [ingress](https://kubernetes.io/ko/docs/concepts/services-networking/ingress/){:target="_blank"}
- [NGINX 인그레스(Ingress) 컨트롤러로 Minikube에서 인그레스 설정하기](https://kubernetes.io/ko/docs/tasks/access-application-cluster/ingress-minikube/){:target="_blank"}
- [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/){:target="_blank"}
- [rinetd 사용법](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=alice_k106&logNo=221091150196){:target="_blank"}
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/){:target="_blank"}
