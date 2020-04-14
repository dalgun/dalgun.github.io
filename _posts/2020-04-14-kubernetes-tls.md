---
layout: post
title: "Kubernetes Ingress Nginx 인증서(TLS) 적용"
subtitle: "Kubernetes Ingress, TLS 쉽게 적용 하기"
author: "Dalgun"
category: "kubernetes"
tag: [Kubernetes, Ingress Nginx, TLS, 인증]
comments: true
---

![thumb](https://www.nginx.com/wp-content/uploads/2020/03/NGINX-Plus-features_Kubernetes-Ingress-Controller.png)


# Kubernetes Ingress Settings
tls 옵션이 없는 상태의 ingress 설정

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dalgun-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: a.b.com
    http:
      paths:
      - path: /
        backend:
          serviceName: gateway-service
          servicePort: 80
      - path: /ui-test/admin
        backend:
          serviceName: admin-ui-service-test
          servicePort: 80
```

# HTTPS 호출 하려고 봤더니

![https-fail](/assets/img/2020-04-14/kuber1.png)

인증서가 적용이 안되어 HTTPS 호출시 오류가 발생한다

# 인증서를 적용해보자

## Secret 생성

```shell script
  kubectl create secret tls ab-tls --key dalgun-key.key --cert dalgun-crt.crt
```

## Ingress Yaml 수정 및 적용

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dalgun-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - a.b.com
    secretName: ab-tls
  rules:
  - host: a.b.com
    http:
      paths:
      - path: /
        backend:
          serviceName: gateway-service
          servicePort: 80
      - path: /ui-test/admin
        backend:
          serviceName: admin-ui-service-test
          servicePort: 80
```


> nginx.ingress.kubernetes.io/ssl-direct: "true" 로 변경시 http 들어오는 호출을 모두 https 로 자동 리다이렉션 한다

# 정상적으로 https 호출 

![https-success](/assets/img/2020-04-14/kuber2.png)



[^1]: 출처:https://www.nginx.com/products/nginx/kubernetes-ingress-controller 