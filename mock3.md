# 1. kubeadm 환경변수 설정
- docs에서 kubeadm 검색 후 "Creating a cluster with kubeadm" 확인
- 해당 페이지의 윗쪽에서 "Container Runtimes" 클릭
- ip_forward부분 복사 후 붙여넣기
- 이후 sysctl로 적용 및 확인
  ```
  > sudo sysctl --system
  > sysctl net.ipv4.ip_forward
    net.ipv4.ip_forward = 1
  ```
(그냥 ___ip_forward___ 검색해도 "Container Runtimes" 나오는것 같기는 함)

# 2. ServiceAccount 및 ClusterRole/ClusterRoleBinding 생성
### service account 생성
```
k create sa pvviewer
```
### ClusterRole/ClusterRoleBinding 생성
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pvviewer-role
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pvviewer-role-binding
subjects:
- kind: ServiceAccount
  name: pvviewer
  namespace: default                # subjects -> ServiceAccount 입력 필요
roleRef:                            # subjects 없어도되는듯! -> 없으면 안됨..!!
  kind: ClusterRole
  name: pvviewer-role
  apiGroup: rbac.authorization.k8s.io
```
**ClusterRoleBinding에 ServiceAccount 꼭 들어가야한다!!!!!!!!!!!!!!**
### service account 포함한 pod 생성
``` yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pvviewer
  name: pvviewer
spec:
  serviceAccountName: pvviewer      # Service Account : pvviewer
  containers:
  - image: redis
    name: pvviewer
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```
# 3. StorageClass 생성
# 4. Configmap 생성 및 Deployment 적용
### Configmap 작성
``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: cm-namespace
data:
  ENV: "production"
  LOG_LEVEL: "info"
```
### deployment.yaml 수정
``` yaml
  containers:
  - envFrom:                # Configmaps에서 envFrom 치면 찾을 수 있음!
    - configMapRef:
        name: app-config
```
# 5. PriorityClass 생성 및 Pod 적용
### PriorityClass 생성
``` yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 50000
description: "description"
```
### Pod 에 적용
``` yaml
spec:
  priorityClassName: low-priority
```
# 6. NetworkPolicy 생성
### 대상 pod의 label 확인
``` yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: np-test-1              # label은 run=np-test-1
```
### 조건에 맞게 NetworkPolicy 작성
``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-nptest
spec:
  podSelector:
    matchLabels:
      run: np-test-1            # label이 run=np-test-1 인 pod 대상
  policyTypes:
  - Ingress                     # Ingress만
  ingress:
  - from:
    - podSelector: {}           # 모든 pod로 부터 들어오는 --> 현재 속한 namespace에 대한 pod만으로 제한된듯..
    ports:
    - protocol: TCP
      port: 80                  # port는 80
```
답안
``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-nptest
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: np-test-1
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
```
# 7. taint 문제
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: prod-redis
    image: redis:alpine
  tolerations:
  - key: "env_type"
    value: "production"
    operator: "Equal"
    effect: "NoSchedule"
```
# 8. PV 조건에 맞는 PVC 붙이기
# 9. kubeconfig 에러
### kubeconfig 에러메세지 확인
```
k get pod --kubeconfig=/root/CKA/super.kubeconfig
E0910 13:36:45.068183   58257 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://controlplane:9999/api?timeout=32s\": dial tcp 192.168.211.140:9999: connect: connection refused"
```
# 10. deployment scale 해도 replicas가 늘어나지않는문제
### deployment scale
```
> k scale deploy nginx-deploy --replicas=3
> k get deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   1/3     1            1           5m9s
```
controller 확인
```
> k describe pod kube-contro1ler-manager-controlplane -nkube-system
  Warning  Failed   2m35s (x5 over 5m29s)  kubelet  Failed to pull image "registry.k8s.io/kube-contro1ler-manager:v1.33.0": rpc error: code = NotFound desc = failed to pull and unpack image "registry.k8s.io/kube-contro1ler-manager:v1.33.0": failed to resolve reference "registry.k8s.io/kube-contro1ler-manager:v1.33.0": registry.k8s.io/kube-contro1ler-manager:v1.33.0: not found
```



