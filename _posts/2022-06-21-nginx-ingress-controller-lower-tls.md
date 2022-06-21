---
layout: post
title: '[K8S] Support TLSv1, TLSv1.1 at Nginx Ingress Controller'
categories: [Development]
tags: [https, tls, ssl, version, k8s, kubernetes, nginx, ingress, controller]
feature_image: 'https://images.unsplash.com/photo-1633265486064-086b219458ec?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2370&q=80'
feature_license: 'Photo by <a href="https://unsplash.com/@towfiqu999999?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Towfiqu barbhuiya</a> on <a href="https://unsplash.com/s/photos/encryption?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>'
use_math: false
published: true
last_modified_at: '2022-06-21 16:13:27'
---

<!-- more -->
> 🖌 Note: 이 포스팅은 Kubernetes community 에서 개발하고 관리하는 "official" NGINX ingress controller 를 기반으로 작성되었습니다. NGINX 에서 제공하는 ingress controller 와는 차이가 있을 수 있습니다.

# 📌 TLS/HTTPS in NGINX Ingress Controller
NGINX ingress controller 의 기본 지원 TLS 버전은 TLSv1.2 / TLSv1.3 이다. 하위 버전의 취약점 때문에 기본 설정에는 빠져 있지만, 오래된 버전의 기기에서도 서비스를 제공하려면 최소한 TLSv1.1 의 지원은 필수적이다.

## 📍 ConfigMap
ConfigMap 설정을 통해 간단히 ingress controller 의 NGINX 서버 설정을 변경해 줄 수 있다.

먼저, ingress controller pod 를 확인하자 (namespace: 'ingress-nginx')
```shell
> kubectl get -n ingress-nginx pods
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-54d8b558d4-798w5   1/1     Running   0          102d
```

다음으로, ingress controller 가 사용하는 configmap 을 확인한다.
```shell
> kubectl describe -n ingress-nginx pod/ingress-nginx-controller-54d8b558d4-798w5 | grep configmap
      --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
```

Configmap 의 이름이 `ingress-nginx-controller` 인 것을 확인했으므로, 아래와 같이 configmap 생성용 yaml 파일을 작성한다.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  allow-snippet-annotations: 'true'
  ssl-protocols: TLSv1 TLSv1.1 TLSv1.2 TLSv1.3
  ssl-ciphers: ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA256:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA
```
{% include code-caption.html text="configmap.yaml" %}
`allow-snippet-annotations` 는 기존 configmap 의 내용을 반영한 것이다. 기존 configmap 에 다른 내용이 있다면 전부 추가해 준다. (`kubectl describe -n ingress-ningx configmap/ingress-nginx-controller` 명령으로 확인)

```shell
> kubectl apply -f configmap.yaml
```
클러스터에 적용하고 나면, 수 초 내지 수 분 이내에 NGINX ingress controller 가 configmap 의 변경을 감지하고 자동으로 변경된 설정을 적용한다. 설정 변경이 적용되었는지는 `kubectl get events -n ingress-nginx` 명령으로 확인할 수 있다.

변경된 설정으로 RELOAD 이벤트가 발생하고 나면 nginx 설정이 변경되었음을 확인하자.
```shell
> kubectl -n ingress-nginx exec -it ingress-nginx-controller-54d8b558d4-798w5 -- cat /etc/nginx/nginx.conf | grep ssl_protocols
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
```

이미 서비스 중인 domain 을 가지고 있다면, https://www.cdn77.com/tls-test 와 같이 서버의 SSL/TLS 지원 여부를 검증해 주는 사이트에서도 확인 가능하다.

# 📌 References
1. [NGINX Ingress Controller - Default TLS Version and Ciphers](https://kubernetes.github.io/ingress-nginx/user-guide/tls/#default-tls-version-and-ciphers)
2. [There are two Nginx Ingress Controllers for k8s. What?](https://grigorkh.medium.com/there-are-two-nginx-ingress-controllers-for-k8s-what-44c7b548e678)