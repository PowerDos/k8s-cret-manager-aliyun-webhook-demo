# 简介
> 这里介绍如何在K8s中通过cret-manager自动创建HTTPS证书，提供两种方式，一种是单域名证书，一种是通过阿里云DNS验证实现通配符域名证书申请

> 我们这里通过Helm安装cret-manager,**请注意查看k8s版本正确安装对应版本的应用**

## 安装Helm 3
> 官方安装教程: https://helm.sh/docs/intro/install/

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

## 安装cert-manager
### 前期准备
#### 添加命名空间
> kubectl create namespace cert-manager

#### 添加cret-manager源
> helm repo add jetstack https://charts.jetstack.io

#### 更新源
> helm repo update

#### 安装CRDs
> 注意安装对应的版本
```
# Kubernetes 1.15+
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager.crds.yaml

# Kubernetes <1.15
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager-legacy.crds.yaml
```

### 安装cert-manager
```
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.15.1
```

### 验证是否安装成功
> 以下结果为成功，你也可以看看镜像日志，是否正常启动，是否正常
```
$ kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6344597-zw8kh               1/1     Running   0          2m
cert-manager-cainjector-348f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-893u48fcdb-nlzsq      1/1     Running   0          2m
```

## 安装证书
> 官方介绍这中 Issuer 与 ClusterIssuer 的概念：

```
Issuers, and ClusterIssuers, are Kubernetes resources that represent certificate authorities (CAs) that are able to generate signed certificates by honoring certificate signing requests. All cert-manager certificates require a referenced issuer that is in a ready condition to attempt to honor the request.
```

> Issuer 与 ClusterIssuer 的区别是 ClusterIssuer 可跨命名空间使用，而 Issuer 需在每个命名空间下配置后才可使用。这里我们使用 ClusterIssuer，其类型选择 Let‘s Encrypt

> 测试证书
```
# cluster-issuer-letsencrypt-staging.yaml

apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # 务必将此处替换为你自己的邮箱, 否则会配置失败。当证书快过期时 Let's Encrypt 会与你联系
    email: gavin.tech@qq.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # 将用来存储 Private Key 的 Secret 资源
      name: letsencrypt-staging
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```

> 正式证书
```
# cluster-issuer-letsencrypt-prod.yaml

apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email:  gavin.tech@qq.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

### 这里安装两个环境的证书的作用
> 这里分别配置了测试环境与生产环境两个 ClusterIssuer， 原因是 Let’s Encrypt 的生产环境有着非常严格的接口调用限制，最好是在测试环境测试通过后，再切换为生产环境。生产环境和测试环境的区别：https://letsencrypt.org/zh-cn/docs/staging-environment/

### 在Ingress中使用证书
> 在ingress配置后，会自动生成证书

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    # 务必添加以下两个注解, 指定 ingress 类型及使用哪个 cluster-issuer
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer："letsencrypt-staging" # 这里先用测试环境的证书测通后，就可以替换成正式服证书

    # 如果你使用 issuer, 使用以下注解 
    # cert-manager.io/issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - example.example.com                # TLS 域名 - 这里仅支持单域名，下面会讲通配符的域名配置
    secretName: quickstart-example-tls   # 用于存储证书的 Secret 对象名字，可以是任意名称，cert-manager会自动生成对应名称的证书名称
  rules:
  - host: example.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```
