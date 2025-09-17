## CKA 문제 정리

### 1. Deployment HPA 설정
1. Kuberentes Doc
    - https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

<br/>

2. kubectl autoscale 명령 사용
    ``` bash
    kubectl autoscale deployment <deployment-name> --cpu-percent=70 --min=2 --max=10
    ```

<br/>

3. stabilizationWindowSeconds 설정
    ``` bash
    # yaml에서 직접 설정 바꿔야 함
    spec:
        behavior:
            scaleDown:
                stabilizationWindowSeconds: 600 // 10분 동안 급격한 축소 방지
            scaleUp:
                stabilizationWindowSeconds: 30 // scale up도 30초 동안 유지 후 올림 
    ```

<br/>

---

<br/>

### 2. CRI 설치
CRI(Container Runtime Interface)는 Kuberentes가 다양한 컨테이너 런타임(Docker, containerd, CRI-O 등)을 사용할 수 있돌고 만든 표준 인터페이스. kubelet은 컨테이너 실행을 직접 하지 않고, CRI를 통해 런타임(containerd, CRI-O 등)에 요청합니다.

mock exam1-2번 문제

1. *.dev 파일 설치, dpkg -i 명령 사용 -> 명령어가 주어짐
    ```bash
    sudo -i # 하면 파일을 못찾음
    sudo dpkg -i <package-file>.deb

    # 전부 sudo 붙이고 동작
    # 서비스 시작
    systemctl start containerd <or crio>

    # 부팅 시 자동 실행
    systemctl enable containerd <or crio>

    # 상태 확인
    systemctl status containerd <or crio>
    or
    systemctl is-active cri-docker
    systemctl is-enabled cri-docker
    ```

<br/>

2. mock exam 1번과 유사한 문제도 같이 나옴
    ```
    net.ipv4.ip_forward 로 doc 검색하면 하는법 나옴
    ```

<br/>

---

<br/>

### 3. CNI 설치

※ Practice Test - Networking CNIs

1. Network Policy가 사용가능한 CNI 설치 ( Flannel과 Calico 중 설치하라고 나와있음. Calico 설치하면 됨. )
    - flannel은 network policy 지원되지 않음
    - Calico는 network policy 지원됨

<br/>

2. curl이나 wget으로 주어진 url 입력해서 파일 받고, 그 파일을 kubectl create -f로 설치
    ``` bash
    # Practice Test - Networking CNIs에서 doc 링크 주어짐 
    # kubectl로 바로 create하게 되어 있는데, 시험 대비해서 wget으로 받기
    # 시험에서 바로 create가 될지 모르기에, wget이 파일로 저장해서 편함

    wget https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/tigera-operator.yaml

    kubectl create -f tigera-operator.yaml

    wget https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/custom-resources.yaml
    
    # cidr 확인하는법
    kubectl cluster-info dump | grep CIDR

    # cidr field 수정
    kubectl create -f custom-resources.yaml

    # 확인
    watch kubectl get pods -A
    or
    watch kubectl get tigerastatus
    ```

<br/>


---
### 4. Sidecar Container - 로깅용도 Sidecar 컨테이너 붙이기
1. 주어지는 문서 page 말고, SideCar로 검색하면 나오는 페이지 활용
2. mockexam 1번 풀기
3. 실수 겁나 많이 함 - 명령어에 bin 붙이면 안되고
    - initContainer로 만들어서 에러남 
    - 기존 pod가 monitor라고 되어 있음, 즉 로그 보는용도
    - 새로 만드는 pod가 로깅 용도
    - 새로 만드는걸 initContainer로 생성하면 기존 pod가 계속 기다려서 에러
    - initContainer, restartPolicy 지워야 함 -> sidecar로 설정
    - 명령어 /bin/sh 이렇게 돼 있는데 'sh', '-c' 하면 동작함

    ```
    # do + 주어진 command
    command: ['sh', '-c', 'while true; do date >> /opt/logs.txt; sleep 1; done']
    ```

---
### 5. Deployment Port 설정 및 Node Port Service 생성
1. kubectl expose 명령어 활용
    ```bash
    # --help / --dry-run=client로 확인 하면서 구현
    kubectl expose deployment <deployment-name> --type=NodePort --port=80
    ```
---
### 6. Deployment Resource 부족으로 Pod 설치 실패 복구
    - Deployment Container의 Resource 수정 -> Container 개수가 2개임!!!
    - Replica 0으로 만들고 3으로 만들어야 제대로 동작
        - replica로 줄이고 안해서 제대로 동작안함ㅠㅠ

---
### 7. Priority Class 생성 후 Deployment에 적용
- User Defined된 Value보다 1작게 설정 (기본으로 system으로 시작 하는 2개의 Priority Class 존재) -> 숫자 세기 주의, 생성 후, 확인해보면 쉬울듯... ㅎㅎ
    ```
    # 리눅스 명령어로 복사 계산
    expr 100000 - 1
    ```
- Deployment 적용 후 적용된 pod만 해당 NS에서 Running됨, 나머지는 무시

---
### 8. StorageClass 생성
    - 주어진 조건대로 Storage Class 생성

---
### 9. 생성되어 있는 PVC를 생성된 PV에 맞춰 수정 후 Deployment에 적용
- PVC와 PV에 mountOption이 맞지 않아 수정필요 -> capacity랑 ReadWriteMany 뭐 이런거 달랐던 것 같음.
- PVC 삭제 후 재생성
- 제공된 Deployment yaml 파일에 PVC 이름 수정 후 적용 필요
- lightning lab 5번 문제와 비슷
- pv와 pvc 비교 
    - accessModes, resource.request.storage, storageclass 확인
- deployment
    - pvc 이름 제대로 들어가 있는지 확인

---
### 10. Cert-Manager 관련 CRD 리스트와 CRD에 명시된 Field를 파일로 저장
- crd 리스트에 label filter 걸어서 출력
- CRD에 명시된 Field의 경우 jsonpath 이용해서 출력 -> root부터 하나씩 json path 출력해가면서 확인해야 편하다고 함...

1. cert-manager 관련 CRD 리스트 확인 (라벨 필터링)
    ```
    # 먼저 라벨 확인
    kubectl get crd -o wide --show-labels | grep cert-manager
    ```

2. 라벨을 확인했다면, 라벨 기준으로 리스트를 뽑기 (라벨 여러개 있음, 골라서 하나만 사용해도 됨)
    ```
    kubectl get crd -l app.kubernetes.io/part-of=cert-manager
    ```

3. default output으로 저장하라고 돼 있길래 name createday? 이 두가지 그냥 파일에 적었던것 같음
    ```
    kubectl get crd -l app.kubernetes.io/part-of=cert-manager > <저장 위치>
    ```

4. 추가 문제 있던거 못풀음 jsonpath?

---

<br/>

### 11. Network Policy 관련
1. 주어진 조건에 맞는 ( Frontend, Backend ns간 통신 ) network policy yaml 적용

<br>

2. 3개의 yaml이 주어지며 셋 중 조건에 맞는 yaml 적용
    - MockExam인가 Network 쪽 Practice 쪽 문제랑 유사
        - mock exam2 - 11번

    ```
    kubectl exec -it frontend -- curl -m 5 172.17.0.5 # 5초 지나면 timeout
    kubectl get networkpolicy
    ```

---
### 12. Config Map 수정 후 Application에 적용, Config Map immutal 설정
- Nginx 관련 Config Map 수정 (수정 필요한 값 명시되어 있음) 후 application에 적용 (재시작) -> application 재시작 중요
- Nginx 관련 Config Map에 immutable 설정 -> configmap immutable 검색하면 나옴
    ```
    immutable: true
    ```
---
### 13. Argo CD Helm 설치
- Argo repo 추가
- 특정 버전의 arco-cd helm template 출력 (CRD 없이) -> helm pull 명령어, --version 지정해서 다운로드 하고, 압축풀어서 (tar -xzf chart.tgz) crd 폴더 삭제 후, 해당 폴더로 설치
- 특정 버젼의 arco-cd helm 설치 (CRD 없이)
- skip-crd 옵션이 먹지 않아, helm fetch로 chart download 후 설치

    ```
    # helm repo 추가
    helm repo add argo https://argoproj.github.io/argo-helm

    # helm pull version 지정 다운로드 <fetch?>
    helm pull argo/argo-cd --version=8.3.5

    # 압축 풀기
    tar --help 검색 후 풀기

    # 압축 푼 template 폴더에 crds 라는 폴더를 제거
    rm -rf template/crds

    # helm template 출력이 무슨 말인지 모르겠음
    목록을 단순히 어디다 저장하면 되는건가? 이걸 무슨 말인지 몰라서 해결 못함

    # crd 없이 argo cd 설치
    helm install <release-name> ./<압축폴더> --namespace <필요할 경우>
    ```

---
### 14. ingress 설정을 Gateway 설정으로 migration
1. 기존 Ingress 설정을 참고해서 Gateway 및 HttpRoute 설정
    - Gateway API 홈페이지의 Guide 문서 참조하면 됨
    - https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/?utm_source=chatgpt.com
    - HTTPS port는 443으로 설정하면 됨
    - TLS 설정보면 나와 있음 (tls 설정 잘못해서 실패함... 잘못된 link 참조)
        ```bash
        spec:
            gatewayClassName: nginx
            listeners:
                ...
                tls:
                    # Terminate로 하면 안되는 것 같음... 주석이 적혀있었네 설정 바꿀것...
                    mode: Terminate # If protocol is `TLS`, `Passthrough` is a possible mode
                    certificateRefs:
                    - name: my-app-cert
                        kind: Secret
        ```
    - 설정완료 후 Ingress resource 삭제 필요
        - 삭제 해야 gateway가 정상동작하는지 확인 가능
        - 혹시 모르니 backup 추천
        - 문제에 적혀 있음
        - 연진님 점수 깎였으니 주의!!!


---
### 15. Trouble Shooting
1. 아무것도 안되는 k8s 살려서 Pod Ready 상태 만들기
2. crictl 명령으로 api-server 죽은거 확인, Log 확인해서 etcd 연결 실패 확인
    ```
    crictl ps | grep kube-apiserver
    crictl logs <container_id>
    ```
3. etcd 설정에 endpoints 확인해서 api-server 값 설정 변경
    ```
    # etcd end-point 확인
    cat /etc/kubernetes/manifests/etcd.yaml | grep listen
    --listen-client-urls=https://127.0.0.1:2379

    # api-server etcd 수정
    vi /etc/kubernetes/manifests/kube-apiserver.yaml
    --etcd-servers=https://127.0.0.1:2379
    ```
4. api-server running 이후 kube-schedule 안삼, crictl에서도 확인 안됨
    ```
    kubectl get pods -n kube-system
    crictl ps
    ```
5. kubelet 로그 확인해서 resource 관련 문제 확인
    ```
    journalctl -u kubelet
    ```
6. kube-schedule의 resource값 및 node의 resource 값 확인 작은 값으로 수정
    - 정 모르겠으면, 다른 cluster 접속해서 파일 설정 비교해보면 됨.
    - systemctl restart kubelet.service 잊지 말기
    ```
    # resource 확인 및 수정
    vi /etc/kubernetes/manifests/kube-scheduler.yaml
    
    resources:
      requests:
        cpu: 100m
    
    systemctl restart kubelet.service
    ```
