## 简介
cert-manager是本地Kubernetes证书管理控制器。它可以帮助从各种来源颁发证书，例如Let's Encrypt， HashiCorp Vault， Venafi，简单的签名密钥对或自签名。

它将确保证书有效并且是最新的，并在到期前尝试在配置的时间续订证书。
![05e7105498ab6b92c8c6ad7bc8383ebf.png](evernotecid://481E08E3-B0BC-4AF0-BD1D-F5A91D581CF3/appyinxiangcom/15709100/ENResource/p946)

## 组件

* [Issuer](https://cert-manager.io/docs/concepts/issuer/) 
* [Certificate](https://cert-manager.io/docs/concepts/certificate/)
* [CertificateRequest](https://cert-manager.io/docs/concepts/certificaterequest/)
* [ACME Orders and Challenges](https://cert-manager.io/docs/concepts/acme-orders-challenges/)
* [Webhook](https://cert-manager.io/docs/concepts/webhook/)
* [CA Injector](https://cert-manager.io/docs/concepts/ca-injector/)

## 安装
创建 CRD
``` sh
# Kubernetes 1.15+
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.2/cert-manager.yaml

# Kubernetes <1.15
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.2/cert-manager-legacy.yaml
```
创建命名空间
```sh
kubectl create namespace cert-manager
```
增加 helm 仓库
```sh
helm repo add jetstack https://charts.jetstack.iov
helm repo update
```
使用 helm 安装 cert-manager
```sh
# Helm v3+
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.14.2

# Helm v2
$ helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.14.2 \
  jetstack/cert-manager
```

## 快速开始
步骤1，部署一个示例应用
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
spec:
  selector:
    matchLabels:
      app: kuard
  replicas: 1
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/rd-pubilc/kuard-amd64:1
        imagePullPolicy: Always
        name: kuard
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: kuard
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: kuard
```
```sh
kubectl apply -f deployment.yaml
```

步骤2，部署一个 ingress：
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    #cert-manager.io/issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - kuard.51charts.com
    secretName: kuard-51charts-tls
  rules:
  - host: kuard.51charts.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```
```sh 
$ kubectl apply -f ingress.yaml
```

测试检查：
```sh
$ curl -kivL -H 'Host: kuard.51charts.com' 'http://192.168.2.195'
```

步骤3， 配置 Let’s Encrypt Issuer
设置两个 issuer, staging-issuer 和 production-issuer, 邮箱要填写有效的地址，用于通知证书过期。因为 Let’s Encrypt 对证书有速率限制，测试使用staging-issuer，验证没问题后更新使用 production-issuer， 两个issuer 配置均使用 HTTP 验证

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
 name: letsencrypt-staging
spec:
 acme:
   # The ACME server URL
   server: https://acme-staging-v02.api.letsencrypt.org/directory
   # Email address used for ACME registration
   email: fanchao@soundbus.cn
   # Name of a secret used to store the ACME account private key
   privateKeySecretRef:
     name: letsencrypt-staging
   # Enable the HTTP-01 challenge provider
   solvers:
   - http01:
       ingress:
         class:  nginx
```
```sh
$ kubectl apply -f staging-issuer.yaml
```

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
 name: letsencrypt-prod
spec:
 acme:
   # The ACME server URL
   server: https://acme-v02.api.letsencrypt.org/directory
   # Email address used for ACME registration
   email: fanchao@soundbus.cn
   # Name of a secret used to store the ACME account private key
   privateKeySecretRef:
     name: letsencrypt-pord
   # Enable the HTTP-01 challenge provider
   solvers:
   - http01:
       ingress:
         class:  nginx
```
```sh
$ kubectl apply -f production-issuer.yaml
```
查看 issuer
```sh
$ kubectl describe issuer letsencrypt-staging
```

```sh
$ kubectl describe issuer letsencrypt-staging

Name:         letsencrypt-staging
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"cert-manager.io/v1alpha2","kind":"Issuer","metadata":{"annotations":{},"name":"letsencrypt-staging","namespace":"default"},...
API Version:  cert-manager.io/v1alpha2
Kind:         Issuer
Metadata:
  Creation Timestamp:  2020-04-26T03:42:00Z
  Generation:          1
  Resource Version:    2012408907
  Self Link:           /apis/cert-manager.io/v1alpha2/namespaces/default/issuers/letsencrypt-staging
  UID:                 e2accf3d-876f-11ea-b349-f22f20f0e290
Spec:
  Acme:
    Email:  fanchao@soundbus.cn
    Private Key Secret Ref:
      Name:  letsencrypt-staging
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Class:  nginx
Status:
  Acme:
    Last Registered Email:  fanchao@soundbus.cn
    Uri:                    https://acme-staging-v02.api.letsencrypt.org/acme/acct/13325827
  Conditions:
    Last Transition Time:  2020-04-26T03:42:02Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none> 
```
应该会看到注册邮箱账户。

更新 ingress，添加在前面的示例中已注释掉的注释，cert-manger 会创建一个certificate,
```sh
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - kuard.51charts.com
    secretName: kuard-51charts-tls
  rules:
  - host: kuard.51charts.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```
```sh
$ kubectl apply -f ingress-staging.yaml
```

```sh
$ kubectl get certificate
NAME                 READY   SECRET               AGE
kuard-51charts-tls   True    kuard-51charts-tls   100m
```
```sh
$ kubectl describe certificate kuard-51charts-tls

Name:         kuard-51charts-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1alpha2
Kind:         Certificate
Metadata:
  Creation Timestamp:  2020-04-27T03:59:26Z
  Generation:          2
  Owner References:
    API Version:           extensions/v1beta1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  kuard
    UID:                   5727822a-883b-11ea-b349-f22f20f0e290
  Resource Version:        2021289797
  Self Link:               /apis/cert-manager.io/v1alpha2/namespaces/default/certificates/kuard-51charts-tls
  UID:                     7c6a442d-883b-11ea-b349-f22f20f0e290
Spec:
  Dns Names:
    kuard.51charts.com
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       Issuer
    Name:       letsencrypt-prod
  Secret Name:  kuard-51charts-tls
Status:
  Conditions:
    Last Transition Time:  2020-04-27T04:01:54Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2020-07-26T03:01:52Z                   <non
Events:
  Type    Reason        Age    From          Message
  ----    ------        ----   ----          -------
  Normal  GeneratedKey  7m21s  cert-manager  Generated a new private key
  Normal  Requested     7m21s  cert-manager  Created new CertificateRequest resource "kuard-51charts-tls-2386411513"
  Normal  Issued        6m43s  cert-manager  Certificate issued successfully
```
证书在几分钟内颁发，完成后，cert-manager将基于入口资源中使用的密钥创建一个包含证书详细信息的密钥。您也可以使用describe命令查看一些详细信息：
```sh
$ kubectl describe secret quickstart-example-tls

Name:         quickstart-example-tls
Namespace:    default
Labels:       <none>
Annotations:  cert-manager.io/alt-names: example.51charts.com
              cert-manager.io/certificate-name: quickstart-example-tls
              cert-manager.io/common-name: example.51charts.com
              cert-manager.io/ip-sans: 
              cert-manager.io/issuer-kind: Issuer
              cert-manager.io/issuer-name: letsencrypt-staging
              cert-manager.io/uri-sans: 

Type:  kubernetes.io/tls

Data
====
tls.crt:  3562 bytes
tls.key:  1679 bytes
ca.crt:   0 bytes
```
现在我们确信一切配置正确，您可以在 ingress 中更新注释以指定生产发行者：
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - example.51charts.com
    secretName: quickstart-example-tls
  rules:
  - host: example.51charts.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```
```sh
kubectl apply -f ingress-prod.yaml
```
您还需要删除 cert-manager 正在监视的现有机密，并使它与更新的发行方一起重新处理请求。
```sh
$ kubectl delete secret kuard-51charts-tls
secret "kuard-51charts-tls" deleted
```
这将开始获取新证书的过程，并使用describe可以查看状态。生产证书更新后，您应该可以在您的域中看到带有签名的TLS证书的示例KUARD。
```sh 
$ kubectl describe certificate

Name:         kuard-51charts-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1alpha2
Kind:         Certificate
Metadata:
  Creation Timestamp:  2020-04-27T03:59:26Z
  Generation:          2
  Owner References:
    API Version:           extensions/v1beta1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  kuard
    UID:                   5727822a-883b-11ea-b349-f22f20f0e290
  Resource Version:        2021289797
  Self Link:               /apis/cert-manager.io/v1alpha2/namespaces/default/certificates/kuard-51charts-tls
  UID:                     7c6a442d-883b-11ea-b349-f22f20f0e290
Spec:
  Dns Names:
    kuard.51charts.com
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       Issuer
    Name:       letsencrypt-prod
  Secret Name:  kuard-51charts-tls
Status:
  Conditions:
    Last Transition Time:  2020-04-27T04:01:54Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2020-07-26T03:01:52Z
Events:
  Type    Reason          Age               From          Message
  ----    ------          ----              ----          -------
  Normal  Requested       50m               cert-manager  Created new CertificateRequest resource "kuard-51charts-tls-4246362355"
  Normal  GeneratedKey    9s (x2 over 50m)  cert-manager  Generated a new private key
  Normal  Requested       9s (x2 over 29m)  cert-manager  Created new CertificateRequest resource "kuard-51charts-tls-2057850725"
  Normal  PrivateKeyLost  9s                cert-manager  Lost private key for CertificateRequest "kuard-51charts-tls-4246362355 ", deleting old resource
  Normal  Issued          4s (x3 over 49m)  cert-manager  Certificate issued successfully
```
至此，https应该可以使用