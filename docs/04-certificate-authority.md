## **04-認証局(CA)のプロビジョニングとTLS証明書の生成**

---

このセクションでは、CloudFlareのPKIツールキットである cfsslを使用してPKI基盤をプロビジョニングします。

PKI基盤を利用して認証局を作り、以下を生成します。
- admin用のTLS証明書
- 各コンポーネントのTLS証明書
  - etcd
  - kube-apiserver
  - kube-controller-manager
  - kube-scheduler
  -  kubelet
  - kube-proxy

> 各コンポーネントの役割は以下にまとまっています。
https://kubernetes.io/docs/concepts/overview/components/

<br>

### **認証局(CA)の立ち上げ**
---

<br>

このステップでは、TLS証明書を生成するための認証局(CA)をプロビジョニングします。

CA設定ファイル、CA自身の証明書、および秘密鍵を生成します。

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

以下のファイルが生成されます。


```
ca-key.pem
ca.pem
```

<br>

### **クライアントとサーバーの証明書発行**
---

<br>

このステップでは、各Kubernetesコンポーネントで使うクライアント証明書とサーバー証明書、Kubernetes `admin` ユーザー用のクライアント証明書を発行します。

<br>

#### **Admin用クライアント証明書**
---

<br>

`admin`用のクライアント証明書と秘密鍵を生成します。

```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

以下のファイルが生成されます。


```
admin-key.pem
admin.pem
```

<br>

#### **Kubeletのクライアント証明書**
---

<br>

Kubernetesは、Node Authorizerと呼ばれる特別な用途向けの認可モードを利用します。

主にKubeletsからのAPIリクエストの認証の役割を担います。

Node Authorizerの認可のためには、`system:nodes group`の中の`system:node:<nodeName>`というユーザー名で認証されるように証明書を作成しなければなりません。

このステップでは、KubernetesワーカーノードごとにNode Authorizerの要求を満たす証明書と秘密鍵を発行します。

```
for i in 0 1 2; do
  instance="worker-${i}"
  instance_hostname="ip-10-0-1-2${i}"
  cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance_hostname}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  internal_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PrivateIpAddress')

  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=${instance_hostname},${external_ip},${internal_ip} \
    -profile=kubernetes \
    worker-${i}-csr.json | cfssljson -bare worker-${i}
done
```

以下のファイルが生成されます。


```

worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

<br>

#### **kube-contorller-manager クライアントの証明書**
---

<br>

`kube-controller-manager`クライアントの証明書と秘密鍵を発行します。

```
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

以下のファイルが生成されます。

```
kube-controller-manager-key.pem
kube-controller-manager.pem
```

<br>

#### **kube-proxy クライアントの証明書**
---

<br>

`kube-proxy`クライアント用に証明書と秘密鍵を発行します。

```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

以下のファイルが生成されます。

```
kube-proxy-key.pem
kube-proxy.pem
```

<br>

#### **kube-scheduler クライアント用の証明書**
---

<br>

kube-scheduler クライアント用の証明書と秘密鍵を発行します。

```
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

以下のファイルが生成されます。

```
kube-scheduler-key.pem
kube-scheduler.pem
```

<br>

#### **Kubernetes API サーバー用証明書**
---

<br>

kubernetes-the-hard-way のstatic IPアドレスは、Kubernetes API サーバーの証明書のSAN(subject alternative names)のリストに含める必要があります。

外部のクライアントでも証明書を使った検証を行うための設定です。

Kubernetes API サーバーの証明書と秘密鍵を生成します。

```
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.0.1.10,10.0.1.11,10.0.1.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

以下のファイルが生成されます。

```
kubernetes-key.pem
kubernetes.pem
```

サービスアカウントのキーペア
[サービスアカウントの管理](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)のドキュメントにあるように、Kubernetes Controller Managerは、サービスアカウントのトークンの生成と署名をするためにキーペアを使用します。

service-account の証明書と秘密鍵を発行します。

```
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

以下のファイルが生成されます。

```
service-account-key.pem
service-account.pem
```

<br>

#### **クライアント証明書とサーバーの証明書の配布**
---

<br>

証明書と秘密鍵をコピーし、各ワーカーインスタンスに配置します。
配置対象は以下です。

- CA証明書
- APIサーバの証明書
- ワーカーノードの証明書と秘密鍵

```
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${external_ip}:~/
done
```

接続処理を続けて良いか聞かれた場合は`yes`を入力します。

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

> 転送に成功すると以下のような出力結果を得られます。

```
ca.pem                                        100% 1318    32.0KB/s   00:00    
worker-1-key.pem                              100% 1675    72.6KB/s   00:00    
worker-1.pem                                  100% 1509    40.6KB/s   00:00 
```

> 上記はワーカーインスタンスの台数分繰り返し処理されます。

コントローラーインスタンスにも同様に配置します。
配置対象は以下です。

- CA証明書
- APIサーバの証明書と秘密鍵
- サービスアカウント払い出し用キーペア

```
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa \
    ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ubuntu@${external_ip}:~/
done
```

接続処理を続けて良いか聞かれた場合は`yes`を入力します。

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

> 転送に成功すると以下のような出力結果を得られます。

```
ca.pem                                        100% 1318    59.2KB/s   00:00    
ca-key.pem                                    100% 1679    42.0KB/s   00:00    
kubernetes-key.pem                            100% 1675    73.4KB/s   00:00    
kubernetes.pem                                100% 1598    44.6KB/s   00:00    
service-account-key.pem                       100% 1675    72.0KB/s   00:00    
service-account.pem                           100% 1440    65.8KB/s   00:00  
```

以下のクライアント証明書は次の手順で使用します。
- kube-proxy
- kube-controller-manager
- kube-scheduler
- kubelet
