# cert-manager-tutorial

## 简介
cert-manager是本地Kubernetes证书管理控制器。它可以帮助从各种来源颁发证书，例如Let's Encrypt， HashiCorp Vault， Venafi，简单的签名密钥对或自签名。

它将确保证书有效并且是最新的，并在到期前尝试在配置的时间续订证书。
![05e7105498ab6b92c8c6ad7bc8383ebf.png](evernotecid://481E08E3-B0BC-4AF0-BD1D-F5A91D581CF3/appyinxiangcom/15709100/ENResource/p946)

## 概念
#### 一，Issuer

* Issuers和ClusterIssuers是代表证书颁发机构（CA）的Kubernetes资源，这些证书颁发机构能够通过兑现证书签名请求来生成签名证书。所有证书管理器证书都需要处于就绪状态的引用发行者，才能尝试满足该请求。
* issuer是带命名空间的资源，如果要创建一个Issuer可以在多个名称空间中使用的单个文件，则应考虑创建ClusterIssuer资源。这几乎与Issuer资源相同，但是没有命名空间，因此可用于Certificates在所有命名空间之间发布

示例：
```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: ca-issuer
  namespace: mesh-system
spec:
  ca:
    secretName: ca-key-pair
```
       
#### 二， Certificate
cert-manager的概念Certificates是定义所需的x509证书，该证书将被更新并保持最新状态。一个 Certificate是一个命名空间资源，它引用Issuer或ClusterIssuer来确定将接受证书请求的内容。
示例：
```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: acme-crt
spec:
  secretName: acme-crt-secret
  dnsNames:
  - foo.example.com
  - bar.example.com
  issuerRef:
    name: letsencrypt-prod
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
    group: cert-manager.io
```
  这Certificate将告诉cert-manager尝试使用名为 letsencrypt-prod 的 Issuer 为foo.example.com 和bar.example.com域获取证书密钥对。如果成功，则生成的密钥和证书将存储在 acme-crt-secret 这个 secret 中。

#### CertificateRequest
CertificateRequest 用于请求从X509证书Issuer。该资源包含PEM编码的证书请求的base64编码的字符串，该字符串被发送到引用的 issuer。成功发行后，将根据证书签名请求返回已签名的证书。CertificateRequests通常由控制器或其他系统使用和管理，除非特别需要，否则我们一般不用去配置
#### ACME Orders and Challenges

#### Webhook

#### CA Injector

#### Project Maturity
