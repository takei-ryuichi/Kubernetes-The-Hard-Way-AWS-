## **07-etcdの起動**
---

<br>

Kubernetesの各コンポーネントはステートレスになっており、クラスタの状態はetcdに格納され管理されています。

つまり、etcdはKubernetesにおいて重要なコンポーネントです。

このセクションでは、3ノードのetcdクラスタを構築して、高可用性と安全な外部アクセスを実現します。

<br>

### **準備**
---

<br>

このステップのコマンドは、`controller-0`、`controller-1`、`controller-2`の各コントローラインスタンスで実行する必要があります。

以下の様にsshコマンドを使用して各コントローラーノードにログインしてください。

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

### **etcdのクラスタメンバーの起動**
---

<br>

> ここからの手順は各コントローラーインスタンスで行います

<br>

#### **etcdのダウンロードとインストール**
---

<br>

etcdのバイナリをgithubからダウンロードします。

```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
```

ダウンロードしたファイルを展開してetcdサーバーとetcdctlのコマンドラインユーティリティを抽出します。

```
tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
```

etcdサーバーの設定を行います。

```
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

インスタンスの内部IPアドレスは、クライアントのリクエストを受け付けて、etcdクラスタ間で通信するために使います。

現在のEC2インスタンスの内部IPアドレスを取得します。

```
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
echo ${INTERNAL_IP}
```

各etcdのメンバーは、etcdクラスター内で名前はユニークにする必要があります。
このチュートリアルでは、現在使用しているEC2インスタンスのホスト名をetcdの名前として設定します。

```
ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
echo "${ETCD_NAME}"
```

etcd.serviceとしてsystemdのユニットファイルを作成します。

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.0.1.10:2380,controller-1=https://10.0.1.11:2380,controller-2=https://10.0.1.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

<br>

### **etcdサーバーの起動**
---

<br>

```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

> ここまでの手順を、各コントローラーノード`controller-0`,`controller-1`,`controller-2`で実行してください。

<br>

### **確認**

<br>

etcdのクラスタメンバーを一覧表示します。

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

>出力例

```
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379, false
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379, false
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379, false
```
