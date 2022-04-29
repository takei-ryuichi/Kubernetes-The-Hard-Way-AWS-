## **09-ワーカーノードの起動**
---

<br>

このセクションでは、3つのKubernetesワーカーノードをプロビジョニングします。

以下のコンポーネントを各ノードにインストールします。
- runc
- gVisor
- container networking plugins
- containerd
- kubelet
- kube-proxy

<br>

### **準備**
---

<br>

この手順で記載されているコマンドは、`worker-0`,`worker-1`、`worker-2`の各ワーカーノードで実行する必要があります。

まずはsshコマンドで各ワーカーノードにログインします

```
for instance in worker-0 worker-1 worker-2; do
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

### Kubernetesのワーカーノードのプロビジョング
---

<br>

使用するライブラリをインストールします。

```
sudo apt-get update
sudo apt-get -y install socat conntrack ipset
```

> `socat`は`kubectl port-forward`コマンドに必要なバイナリです。

<br>

#### **swapの無効化**
---

<br>

swapが有効になっている場合、kubeletの起動に失敗します。

以下のコマンドでswapが有効になっているかどうかを確認します。

```
sudo swapon --show
```

出力が何もない場合はswapは無効化されています。

有効になっている場合は以下のコマンドでswapを無効化します。

```
sudo swapoff -a
```

ワーカーのバイナリをダウンロードしてインストールします。

```
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.21.0/crictl-v1.21.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz \
  https://github.com/containerd/containerd/releases/download/v1.4.4/containerd-1.4.4-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubelet
```

インストールする先のディレクトリを作成します。

```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

ワーカーのバイナリを展開します。

```
mkdir containerd
tar -xvf crictl-v1.21.0-linux-amd64.tar.gz
tar -xvf containerd-1.4.4-linux-amd64.tar.gz -C containerd
sudo tar -xvf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
sudo mv runc.amd64 runc
chmod +x crictl kubectl kube-proxy kubelet runc 
sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
sudo mv containerd/bin/* /bin/
```

<br>

### **CNIネットワーキングの設定**
---

<br>

現在のEC2インスタンスのPodに設定されているCIDR範囲を取得します。

```
POD_CIDR=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^pod-cidr" | cut -d"=" -f2)
echo "${POD_CIDR}"
```

bridgeネットワークの設定ファイルを作ります。

```
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.4.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

loopbackネットワークの設定ファイルを作ります。
```
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.4.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```

<br>

### **containerdの設定**
---

<br>

`containerd`の設定ファイルを作成します。

```
sudo mkdir -p /etc/containerd/
```

```
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
```

`containerd.service`systemdのユニットファイルを作成します。

```
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

<br>

### **Kubeletの設定**
---

<br>

```
WORKER_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
| tr "|" "\n" | grep "^name" | cut -d"=" -f2)
echo "${WORKER_NAME}"

sudo mv ${WORKER_NAME}-key.pem ${WORKER_NAME}.pem /var/lib/kubelet/
sudo mv ${WORKER_NAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/
```

kubelet-config.yaml設定ファイルを作ります。

```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${WORKER_NAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${WORKER_NAME}-key.pem"
EOF
```

> `resolvconf`は`systemd-resolv`を使用してCoreDNSを実行するときにループを回避するために使用されます。

`kubelet.servicesystemd`ユニットファイルを作成します。

```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

```

<br>

### **Kubernetes Proxyの設定**
---

<br>

```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

`kube-proxy-config.yaml` 設定ファイルを作成します。

```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

`kube-proxy.service`systemdユニットファイルを作成します。

```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

<br>

### **ワーカーのサービス群の起動**
---

<br>

```
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

> ここまでの手順を各ワーカーノード、`worker-0`、`worker-1`、`worker-2`で実行します。

<br>

### **確認**
---

<br>

> 以下のコマンドは、自身のMacもしくはLinux端末から実行します。

登録されているKubernetesノードの一覧を表示させます。

```
external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=controller-0" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

ssh -i kubernetes.id_rsa ubuntu@${external_ip} kubectl get nodes --kubeconfig admin.kubeconfig
```

> 出力例

```
NAME           STATUS   ROLES    AGE   VERSION
ip-10-0-1-20   Ready    <none>   32s   v1.21.0
ip-10-0-1-21   Ready    <none>   29s   v1.21.0
ip-10-0-1-22   Ready    <none>   26s   v1.21.0
```
