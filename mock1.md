# 1. Container가 3개인 Pod 생성 문제
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: mc-pod
  namespace: mc-namespace
spec:
  volumes:                              # volumes 들어가면 있음
  - name: shared
    emptyDir: {}
  containers:
  - name: mc-pod-1
    image: nginx:1-alpine
    env:
      - name: NODE_NAME
        valueFrom:                      # pod로 검색해서 "Expose Pod Information to Containers ... " 들어가면 있음
          fieldRef:
            fieldPath: spec.nodeName
  - name: mc-pod-2
    image: busybox:1
    command: ['sh', '-c', 'while(true); do date >> /var/log/shared/date.log; sleep 1; done']
    volumeMounts:
    - mountPath: /var/log/shared
      name: shared
  - name: mc-pod-3
    image: busybox:1
    command: ['sh', '-c', 'tail -f /var/log/shared/date.log']
    volumeMounts:
    - mountPath: /var/log/shared
      name: shared
```

# 2. CRI 설치 문제
```
> dpkg -i cri-docker_0.3.16.3-0.debian.deb
> systemctl start cri-docker
> systemctl enable cri-docker
> systemsctl status cri-docker
```

# 3. VPA관련 CRD 출력 문제
```
> k get crd |grep vertical
or
> k get crd --show-labels | grep vertical
```

# 4. 운용중인 Pod를 Service로 expose 하는 문제
```
k expose pod
```

# 5. Deployment 생성 문제
# 6. Container가 살아나지않는 pod troubleshooting
```
k describe pod
```
# 7. 배포한 deployment를 Service로 expose 하는 문제
```
k expose deploy <deployment> --name=<serviceName> --port=8080 --type=NodePort
```
# 8. pv 생성문제
``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
spec:
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  hostPath:
    path: "/pv/data-analytics"
```
# 9. HPA 생성
``` yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kkapp-deploy
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
      stabilizationWindowSeconds: 300
```
# 10. VPA 생성
``` yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: analytics-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:       analytics-deployment
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources: ["cpu", "memory"]
```
# 11. Gateway 생성
``` yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```
# 12. helm upgrade
```
> helm list -A
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
kk-mock1        kk-ns           1               2025-09-11 12:27:59.576206109 +0000 UTC deployed        nginx-18.1.0    1.27.0 

> helm repo list
NAME            URL                               
kk-mock1        https://charts.bitnami.com/bitnami

> helm search repo kk-mock1 |grep nginx
kk-mock1/nginx                                  21.1.23         1.29.1          NGINX Open Source is a web server that can be a...

> helm search repo kk-mock1/nginx --versions |grep 18.1.15
kk-mock1/nginx                          18.1.15         1.27.1          NGINX Open Source is a web server that can be a...

> helm upgrade kk-mock1 -nkk-ns kk-mock1/nginx --version 18.1.15
> helm list -A
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
kk-mock1        kk-ns           2               2025-09-11 12:31:09.864227936 +0000 UTC deployed        nginx-18.1.15   1.27.1 
```

