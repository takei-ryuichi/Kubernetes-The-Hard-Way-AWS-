
## **08-Kubernetesコントロールプレーンの起動**
---

<br>

このセクションでは、3つのインスタンスを使って可用性の高いKubernetesコントロールプレーンを作成します。

合わせて、Kubernetes API Serverを外部クライアントに公開する外部ロードバランサーも作成します。

各ノードに、Kubernetes API Server、Scheduler、およびController Managerのコンポーネントをインストールします。

<br>

### **前提条件**
---


#### **事前確認**
---

<br>

各コントローラノードにログインする前に、自身の端末でKubernetesクラスタの外部公開用のFQDNを確認する必要があります。

以下のコマンドを実行し、FQDNを確認します。

```
 echo ${KUBERNETES_PUBLIC_ADDRESS}
```
> 後続のステップで使用するため、出力結果をメモしてください。


<br>

#### **作業対象**
---

<br>

前のステップと同様、このステップでも`controller-0`,`controller-1`,`controller-2`の各コントローラインスタンスで実行する必要があります。

全てのコントローラーノードにsshコマンドでログインしてコマンドを実行します

> 既に各コントローラーノードにログインしている状態であれば、次の「Kubernetesコントロールプレーンのプロビジョニング」に飛んでください

```
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  echo ssh -i kubernetes.id_rsa ubuntu@$external_ip
done
```



> ここからの手順は、直前のコマンドによって出力されたそれぞれのIPアドレスにssh接続して行います。
（3台のインスタンス全てで同じコマンドを実行する必要があります）

<br>

#### **tmuxを使ったパラレルなコマンド実行**
---

<br>

tmuxを使えば、容易に複数のインスタンスで同時にコマンドを実行できます。
詳細は[こちら](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/01-prerequisites.md#running-commands-in-parallel-with-tmux)をご覧ください.

<br>

### **Kubernetesコントロールプレーンのプロビジョニング**
---

<br>

Kubernetesのconfigファイルを配置するためのディレクトリを作成します。

```
sudo mkdir -p /etc/kubernetes/config
```

<br>

#### **KubernetesコントローラーのバイナリのDLとインストール**
---

<br>

Kubernetesの公式のリリースバイナリをダウンロードします

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl"
```

ダウンロードしたバイナリを移動させます。

```
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

<br>

#### **KubernetesAPIサーバーの設定**
---

<br>

```
sudo mkdir -p /var/lib/kubernetes/

sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
```

APIサーバーをクラスターのメンバーに知らせるための設定として、インスタンスの内部IPアドレスを使います。

現在のEC2インスタンスの内部IPアドレスを取得します。

```
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
echo ${INTERNAL_IP}
```


kube-apiserver.serviceのsystemdのユニットファイルを生成します。

```
KUBERNETES_PUBLIC_ADDRESS=[事前確認の出力結果](https://github.com/takei-ryuichi/Kubernetes-The-Hard-Way-AWS-/blob/main/docs/15-cleanup.md)
```

```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.0.1.10:2379,https://10.0.1.11:2379,https://10.0.1.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${KUBERNETES_PUBLIC_ADDRESS}:443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

参考：https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/

<br>

#### **Kubernetesのコントローラーマネージャーの設定**
---

<br>

`kube-controller-manager`のkubeconfigを移動させます。

```
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

`kube-controller-manager.service`のsystemdユニットファイルを生成します。

```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

<br>

#### **Kubernetesのschedulerの設定**
---

<br>

```
sudo mkdir -p /etc/kubernetes/config/
```

`kube-scheduler`の`kubeconfig`を移動させます。

```
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

`kube-scheduler.yaml`を作ります。

```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

`kube-scheduler.service`のsystemdユニットファイルを生成します。

```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

<br>

#### **コントローラーサービスの起動**
---

<br>

```
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

Kubernetes APIサーバーは初期化が完了するまで30秒くらいかかります。

<br>

#### **確認**
---

<br>

コントローラの各コンポーネントの状態を確認します。

```
kubectl cluster-info --kubeconfig admin.kubeconfig
```

> 出力例

```
Kubernetes control plane is running at https://127.0.0.1:6443
```

<br>

### **ホストファイルのエントリー追加**
---

<br>

`kubectl exec`コマンドを実行する際、コントローラノードはそれぞれのワーカーノードの名前解決を行う必要があります。

ホストインスタンスのhostsファイルに、ワーカーノードのホストエントリーを手動で追加します。

```
cat <<EOF | sudo tee -a /etc/hosts
10.0.1.20 ip-10-0-1-20
10.0.1.21 ip-10-0-1-21
10.0.1.22 ip-10-0-1-22
EOF
```
> この手順を実行しないと、後続のテストが失敗します。

<br>

### **kubelet認証のRBAC**
---

<br>

このステップでは、Kubernetes APIサーバーが各ワーカーノードのKubelet APIにアクセスできるようにRBACによるアクセス許可を設定します。

メトリックやログの取得、Pod内でのコマンドの実行には、Kubernetes APIサーバーからKubelet APIへのアクセスが必要です。

> このチュートリアルでは、`Kubelet ---authorization-mode`フラグをWebhookに設定します。
Webhookモードは SubjectAccessReview APIを使用して認証を行います

ここで実行する以下のコマンドはクラスター全体に作用します。このため適当なコントローラーノードにログインして一度だけ実行してください。
以下では念のため、デプロイ用インスタンスから`controller-0`ノードにログインする所から手順を記載しています。

```
external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=controller-0" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

ssh -i kubernetes.id_rsa ubuntu@${external_ip}
kube-apiserver-to-kubeletという名前でClusterRoleを作ります。
```

このロールに、Kubelet APIにアクセスしてポッドの管理に関連するタスクを実行する権限を付与します。

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

Kubernetes APIサーバーは、`--kubelet-client-certificate`フラグで定義したクライアント証明書を使って、`kubernetes`ユーザーとして`Kubelet`に対して認証を行います。

`system:kube-apiserver-to-kubelet`の`ClusterRole`を`kubernetes`ユーザーにバインドします。

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

<br>

### **Kubernetesクラスターのパブリックエンドポイントを有効化する**
---

<br>

以下のコマンドは、EC2インスタンスではなく自身のMacもしくはLinux環境のラップトップで行います。

`kubernetes-the-hard-way`ロードバランサーのアドレスを取得します。

```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[].DNSName')
echo ${KUBERNETES_PUBLIC_ADDRESS}
```

HTTPリクエストを作成しKubernetesのVersion情報を取得します。

```
curl -k --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}/version
```

> 出力例

```
{
"major": "1",
"minor": "21",
"gitVersion": "v1.21.0",
"gitCommit": "cb303e613a121a29364f75cc67d3d580833a7479",
"gitTreeState": "clean",
"buildDate": "2021-04-08T16:25:06Z",
"goVersion": "go1.16.1",
"compiler": "gc",
"platform": "linux/amd64"
}
```
