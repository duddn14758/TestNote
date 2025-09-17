# 1. Stroage Class
``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-sc ## name
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  ## default class
provisioner: kubernetes.io/no-provisioner  ## provisional
allowVolumeExpansion: true ## VolumeExpansion
volumeBindingMode: WaitForFirstConsumer
```

# 2. Sidecar Container
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logging-deployment  # name
  namespace: logging-ns     # namespace
spec:
  replicas: 1               # replicas
  selector:
    matchLabels:
      app: nginx            # spec.selector.matchLabels 필수!
  template:
    metadata:
      labels:
        app: nginx          # spec.template.metadata.labels 필수!
    spec:
      containers:               # container
        - name: app-container   # container name
          image: busybox        # image
          command: ['sh', '-c', "while true; do echo 'Log entry' >> /var/log/app/app.log; sleep 5; done"]
          volumeMounts:
            - name: data        #volume name
              mountPath: /var/log/app   # mountPath
      initContainers:
        - name: log-agent       
          image: busybox
          restartPolicy: Always
          command: ['sh', '-c', 'tail -f /var/log/app/app.log']
          volumeMounts:
            - name: data
              mountPath: /var/log/app
      volumes:
        - name: data
          emptyDir: {}
```
-> **sidecar container 작성 시 누가 init Container에 들어가야하는지 확인 필요!!** \
--> sidecar로 작성하라고 한게 initContainers에 들어가는게 맞는듯!!\
---> restartPolicy 빼먹으면 안되고, 일단 한번 구동하면 Running까지 세번정도 restart 하는듯
```
> k get pod
NAME                                  READY   STATUS    RESTARTS      AGE
logging-deployment-6749b7877f-tt9sm   2/2     Running   3 (84s ago)   101s
```

# 3. Ingress 만들기
http/https 차이점과 proxy 에 대해 다시한번 확인해보기
``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: ingress-ns
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: "kodekloud-ingress.app"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
```

# 4. deployment 만들고 버전 업그레이드하기
# 5. CertificateSigningRequest
### 5.1 CSR 작성
``` yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-developer # csr name
spec:
  request: ...
    signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```
### 5.2 csr 파싱 및 복사
```
controlplane ~ ➜  cat /root/CKA/john.csr |base64 |tr -d '\n'
```
### 5.3 role/rolebinding작성
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]  # can be pods, serviceaccounts etc...
  verbs: ["get", "create", "list", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: role-binding
  namespace: development
subjects:
- kind: User
  name: john
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer   # above role name
  apiGroup: rbac.authorization.k8s.io
```
다하고 approve 해야한다!
```
k certificate approve john-developer
```
# 6. Pod 생성 및 Service expose
```
k run nginx-resolver --image=nginx

k expose pod nginx-resolver --name=nginx-resolver-service --port=80

k run test-pod --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-resolver-service
Server:    172.20.0.10
Address 1: 172.20.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-resolver-service
Address 1: 172.20.227.118 nginx-resolver-service.default.svc.cluster.local
pod "test-pod" deleted

k run test-pod --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-resolver-service >> /root/CKA/nginx.svc
k run test-pod --image=busybox:1.28 --rm -it --restart=Never -- nslookup 172-20-227-118.default.pod  >> /root/CKA/nginx.pod
```
# 7. Create static pod
```
/etc/kubernetes/manifests/
```
# 8. HPA 생성 문제
``` yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deployment
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource          # type은 언제나 바뀔수있음
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 65
```
# 9. Gateway TLS certification 추가 문제
``` yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: cka5673
spec:
  gatewayClassName: nginx       # gateway class 확인
  listeners:
  - allowedRoutes:
      namespaces:
        from: Same
    name: http
    port: 80
    protocol: HTTP
  - name: foo-https             # 기존 존재하던 http 아래에더 https만 더 붙여주기
    protocol: HTTPS
    port: 443
    hostname: kodekloud.com
    tls:
      certificateRefs:
      - kind: Secret
        group: ""
        name: kodekloud-tls
```

``` yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: cka5673
spec:
  gatewayClassName: kodekloud
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: kodekloud.com
      tls:
        certificateRefs:
          - name: kodekloud-tls
```

# 10. 잘못 배포된 helm 이미지 삭제

# 11. 기준에 맞는 NetworkPolicy 적용시키기
- frontend namespace app -> backend namespace app (O)
- database namespace app -> backend namespace app (X)
``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: net-policy-1
  namespace: backend        # target namespace
spec:
  podSelector: {}           # podSelector 가 {} 면 모든 pod가 대상
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          access: allowed       # namespace 중 label이 access: allowed 인 애만 허용
    ports:
    - protocol: TCP
      port: 80
```
``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: net-policy-2
  namespace: backend
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend        # frontend namespace에 있는것들로부터
    - namespaceSelector:
        matchLabels:
          name: databases       # backend namespace에 있는것들로부터
    ports:
    - protocol: TCP
      port: 80
```
``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: net-policy-3
  namespace: backend
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend        # frontend namespace에 있는것들
    ports:
    - protocol: TCP
      port: 80
```

## Network Policy 관련 확인

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default            # 적용범위
spec:
  podSelector:
    matchLabels:
      role: db                  # defaultnamespace중 라벨이 role=db인 애만 적용 (target인듯)
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16     # 172.17.0.0/16 대역 허용
        except:                 # 172.17.1.0/24 대역 거부
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject    # 라벨이 project=myproject 인 namespace에 대해 허용 
    - podSelector:
        matchLabels:
          role: frontend        # 같은 namespace내에서 라벨이 role=frontend인 애들 허용
    ports:
    - protocol: TCP
      port: 6379                # 위 허용된 소스들이 db pod 에 접근할때는 6379 port만 허용
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24       # 10.0.0.0/24 대역만 허용
    ports:
    - protocol: TCP
      port: 5978                # 나가는 traffic은 5978만 허용
```