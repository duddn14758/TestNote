# Gateway API 구조
### Gateway
"진입지점" 자체\
(LoadBalancer, NodePort, ClusterIP 등으로 열리는 실제 네트워크 엔드포인트)
### Route (HTTPRoute)
들어온 요청을 실제 Service로 어떻게 전달할지 정의하는 규칙 집합

**Ingress -> Gateway로 바꾸려면 대부분 두개 다 필요하다!**

# Example
### 원본 Ingress.yaml
``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    some-ingress-controller.example.org/tls-redirect: "True"
spec:
  ingressClassName: prod
  tls:
  - hosts:
    - foo.example.com
    - bar.example.com
    secretName: example-com
  rules:
  - host: foo.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: foo-app
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: foo-orders-app
            port:
              number: 80
  - host: bar.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bar-app
            port:
              number: 80
```

### Gateway
``` yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: prod
  listeners:
  - name: http                    # 기존 http는 그대로 유지
    port: 80
    protocol: HTTP
    hostname: "*.example.com"
  - name: https                   # ingress에서는 없었지만 gateway로 넘어오면서 tls는 https로 변경
    port: 443                     # https는 443포트로 픽스
    protocol: HTTPS
    hostname: "*.example.com"
    tls:                          # TLS 설정
      mode: Terminate
      certificateRefs:            # secret ref
      - kind: Secret
        name: example-com
  - name: https-default-tls-mode
    port: 8443
    protocol: HTTPS
    hostname: "*.foo.com"
    tls:
      certificateRefs:
      - kind: Secret
        name: foo-com
```
Gateway는 진입지점이기때문에 service와 연결할 필요가 없다\
tls 모드만 정해주면됨\
**HTTPS 관련 설정은 Gateway에서 다 한다**

### HTTPRoute (foo)
위 Ingress에서 host가 두개이기 때문에 HTTPRoute는 두개로 나눠진다.
``` yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: foo
spec:
  parentRefs:
  - name: example-gateway
    sectionName: https
  hostnames:
  - foo.example.com             # Ingress의 hosts
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: foo-app             # Service 이름
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /orders
    backendRefs:
    - name: foo-orders-app
      port: 80
```
ingress의 rules.http.paths 가 gateway의 rules.matches에 매핑되는듯\
backendRefs에 service name이 들어가는듯

### HTTPRoute (bar)
``` yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bar
spec:
  parentRefs:
  - name: example-gateway
    sectionName: https
  hostnames:
  - bar.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: bar-app
      port: 80
```