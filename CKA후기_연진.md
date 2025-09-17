# 1. Deployment HPA 설정
- kubectl autoscale 명령 사용
- stabilizationWindowSeconds 설정
``` yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # AutoScaling page에 있음
```

# 2. CRI설치
- *deb 파일 설치. dpkg -i 명령 사용 -> 명령어가 주어짐
- sysctl 설정 ( mock exam 문제와 유사 )
```
sudo dpkg -i ./docker-cri
systemctl start docker-cri
systemctl enable docker-cri
```

# 3. CNI 설치
- Network Policy가 사용가능한 CNI 설치 ( Flannel과 Calico 중 설치하라고 나와있음)
- Calico 설치하면 됨. (curl 이나 wget으로 주어진 url 입력해서 파일 받고, 그 파일을 kubectl create -f 로 설치)
```
wget http://calico/io
kubectl create -f calico
```

# 4. Sidecar Container
- 주어지는 문서 page 말고 sidecar로 검색하면 나오는 페이지 활용 (command 입력 주의)
``` yaml
spec:
  containers:
  - env:
    - name: DEMO_GREETING
      value: "Hello from the environment"
    - name: DEMO_FAREWELL
      value: "Such a sweet sorrow"
    name: envar-demo-container
    image: gcr.io/google-samples/hello-app:2.0
```
이런식으로 되어있다고 쫄지말자!\
env랑 name,image랑 같은계위니까 침착하게 찾아보자\
sidecar 가 보통 initContainers에 들어가는거같음. command 보고 대충 파악해도될듯

# 5. Deployment Port 설정 및 Node Port Service 생성
```
k expose deploy my-app --name=my-svc --port=80 --type=NodePort
```
이런식으로 조회가 될것
# 6. Deployment Resource 부족으로 Pod 설치 실패 복구
- Deployment container의 Resource 수정 -> Container 갯수가 2개임!
- Replica 0으로 만들고 3으로 만들어야 제대로 동작
```
kubectl describe pod webapp-7c89f4d6b-klmno
...
Events:
  Warning  FailedScheduling  <invalid>  default-scheduler  0/1 nodes are available: 1 Insufficient cpu.
```
``` yaml
resources:
    requests:
        cpu: "500m"
        memory: "512Mi"
    limits:
        cpu: "1000m"
        memory: "1Gi"
```
```
kubectl describe node controlplane | grep -A5 Allocatable
Allocatable:
  cpu:     1000m
  memory:  65735880Ki
  pods:    110
```
-> 이렇게되면 지금 pod는 2개밖에 못사는것\
-> 2ndweek.md page에 자세하게 정리해놓았음으로 command만 써두겠음
# 7. PriorityClass 생성 후 Deployment에 적용
- user defined 된 Value 보다 1 작게 설정 (기본으로 system으로 시작하는 2개의 Priority Class 존재 -> 숫자세기 주의, 생성후 확인해보면 쉬울듯..)
- Deployment 적용 후 적용된 Pod만 해당 ns에서 Running됨. 나머지는 무시
```
controlplane ~ ➜  k get priorityclass
NAME                      VALUE        GLOBAL-DEFAULT   AGE     PREEMPTIONPOLICY
system-cluster-critical   2000000000   false            8m59s   PreemptLowerPriority
system-node-critical      2000001000   false            8m59s   PreemptLowerPriority
```
``` yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```
``` yaml
priorityClassName: high-priority    # pod에도 적용해야함!
```
# 8. StorageClass 생성
- 주어진 조건대로 Storage Class 생성
- 생성되어있는 PVC를 생성된 PV에 맞춰 수정 후 적용 필요
``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  storageClassName: local-sc
  volumeName: local-pv
  resources:
    requests:
      storage: 5Gi
```

# 9. Cert-Manager 관련 CRD 리스트와 CRD에 명시된 Field를 파일로 저장
- CRD리스트에 label filter 걸어서 출력
- CRD에 명시된 Field의 경우 jsonpath 이용해서 출력 -> root부터 하나씩 json path 출력해가면서 확인해야 편하다고함..
```
controlplane ~ ➜  k get crd --show-labels
NAME                                                  CREATED AT             LABELS
blockaffinities.crd.projectcalico.org                 2025-09-09T15:17:17Z   <none>
caliconodestatuses.crd.projectcalico.org              2025-09-09T15:17:18Z   <none>
clientsettingspolicies.gateway.nginx.org              2025-09-09T15:25:37Z   gateway.networking.k8s.io/policy=inherited
```
```
k get crd -o jsonpath='{.items[0]}
k get crd -o jsonpath='{.items[0].spec}'
k get crd -o jsonpath='{.items[0].spec.names}'
k get crd -o jsonpath='{.items[0].spec.names.plural}'
k get crd -o jsonpath='{.items[*].spec.names.plural}'
k get crd -o jsonpath='{.items[*].spec.names.plural}'| tr -s ' ' '\n'
adminnetworkpolicies
bgpconfigurations
bgpfilters
bgppeers
blockaffinities
caliconodestatuses
clientsettingspolicies
clusterinformations
felixconfigurations
...
```
# 10. Network Policy 관련
- 주어진 조건에 맞는 ( Frontend, backend ns 간 통신) network policy yaml 적용
- 3개의 yaml이 주어지며 셋 중 조건에 맞는 yaml 적용

-> **mock2 11번 문항 참고!**

# 12. Configmap 수정 후 Application에 적용. Configmap immutable 설정
- Nginx 관련 Configmap 에 immutable 설정 -> configmap immutable 검색하면 나옴
``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
immutable: true
```
적용 : envFrom(Configmap 문서), valueFrom(environment)
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-fieldref
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox:1.27.2
      envFrom:                      # Configmap 문서
        - configMapRef:
            name: myconfigmap
      env:
        - name: MY_NODE_NAME
          valueFrom:                # environment 문서
            fieldRef:
              fieldPath: spec.nodeName
```

# 13. Argo CD helm 설치
- Argo repo 추가
- 특정 버전의 arco-cd helm template 출력 (CRD 없이) 
  -> helm pull 명령어, --version 지정해서 다운로드하고, 압축 풀어서( tar -xzf chart.tgz) crd 폴더 삭제 후 해당폴더로 설치
- skip-crd 옵션이 먹지 않아, helm fetch로 chart download후 설치
- helm command 잘 활용하기
```
> helm repo add argo https://argoproj.github.io/argo-helm

> helm search repo argo
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
argo/argo                       1.0.0           v2.12.5         A Helm chart for Argo Workflows                   
argo/argo-cd                    8.3.5           v3.1.4          A Helm chart for Argo CD, a declarative, GitOps...
argo/argo-ci                    1.0.0           v1.0.0-alpha2   A Helm chart for Argo-CI                          

> helm search repo argo/argo-cd --versions
argo/argo-cd    8.3.5           v3.1.4          A Helm chart for Argo CD, a declarative, GitOps...
argo/argo-cd    8.3.4           v3.1.3          A Helm chart for Argo CD, a declarative, GitOps...
argo/argo-cd    8.3.3           v3.1.1          A Helm chart for Argo CD, a declarative, GitOps...

> helm pull argo/argo-cd --version 8.3.0
> ls
argo-cd-8.3.0.tgz

> tar -xzf argo-cd-8.3.0.tgz
> cd argo-cd/
> rm -rf templates/crds
> helm template argocd . -n argo-ns |grep namespace
  namespace: argo-ns
  namespace: argo-ns
  namespace: argo-ns
```

# 14. ingress 설정을 Gateway 설정으로 migration
- 기존 ingress  설정을 참고해서 Gateway 및 HttpRoute설정
- HTTPS port는 443으로 설정하면됨
- Gateway API 홈페이지의 Guide 문서 참조하면 됨 -> TLS 설정 보면 나와있음.
- 설정 완료 후 Ingress resource 삭제 필요 (혹시모르니 backup!)

**GatewayApi.md 참고~~!**

# 15. Trouble shooting
- 아무것도 안되는 k8s 살려서 Pod Ready 상태 만들기
```
> k get pod
The connection to the server controlplane:6443 was refused - did you specify the right host or port?
```
- crictl 명령으로 api-server 죽은거 확인. Log 확인해서 etcd 연결 실패 확인
```
> crictl ps -a
CONTAINER           IMAGE               CREATED              STATE               NAME                      ATTEMPT             POD ID              POD                                        NAMESPACE
70a5b2f52927e       6ba9545b2183e       30 seconds ago       Exited              kube-apiserver               1                   7d8715f43f25d       kube-apiserver-controlplane                kube-system
```
- etcd 설정에 endpoints 확인해서 api-server 값 설정 변경 ( /etc/kubernetes/manifest/kube-apiserver.yaml )
```
> cat etcd.yaml |grep listen
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.27.138:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://192.168.27.138:2380
> systemctl restart kubelet     # kubelet 항상 재시작 해줘야함!
```
- api-server running 이후 kube-schedule 안삼, crictl에서도 확인 안됨
```
> k get pod -A
NAMESPACE       NAME                                       READY   STATUS    RESTARTS   AGE
kube-system     kube-scheduler-controlplane                0/1     Pending   0          9m41s
```

- kubelet 로그 확인해서 resource 관련 문제 확인
```
journalctl -u kubelet
```
- kube-scheduler의 resource 값 및 node의 resource 값 확인. 작은값으로 수정
  ( 정 모르겠으면 다른 cluster 접속해서 파일설정 비교해보면 됨!)
  systemctl restart kubelet.service 잊지말기!
```
cat /etc/kubernetes/manifests/kube-scheduler.yaml |grep request -A1
      requests:
        cpu: 100m

> systemctl restart kubelet
```
