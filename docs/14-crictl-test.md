## **14-crictlを使用してワーカーノードのイメージ・ポッド・コンテナをチェックする**
---

<br>

このセクションは、ワーカーノードにログインし、リソース一覧を確認します。

このセクションで扱うコマンドは、立ち上がっている3台のワーカーノード全てで実行可能です。


```
external_ip=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=worker-0" \
  --output text --query 'Reservations[].Instances[].PublicIpAddress')
```
```
ssh -i kubernetes.id_rsa ubuntu@${external_ip}
```

以下のコマンドを実行し、出力を確認します

```
sudo crictl -r unix:///var/run/containerd/containerd.sock images
```

> 出力例

```
IMAGE                                                  TAG                 IMAGE ID            SIZE
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64   1.14.7              5feec37454f45       10.9MB
gcr.io/google_containers/k8s-dns-kube-dns-amd64        1.14.7              5d049a8c4eec9       13.1MB
gcr.io/google_containers/k8s-dns-sidecar-amd64         1.14.7              db76ee297b859       11.2MB
k8s.gcr.io/pause                                       3.1                 da86e6ba6ca19       317kB
```

```
sudo crictl -r unix:///var/run/containerd/containerd.sock pods
```

> 出力例

```
POD ID              CREATED             STATE               NAME                        NAMESPACE           ATTEMPT
9a304a19557f7       2 hours ago         Ready               kube-dns-864b8bdc77-c5vc2   kube-system         0
```

```
sudo crictl -r unix:///var/run/containerd/containerd.sock ps
```

> 出力例

```
CONTAINER ID        IMAGE                                                                     CREATED             STATE               NAME                ATTEMPT
611bfea53997d       sha256:db76ee297b8597fc007b23a90619314b8405bb1df6dcad189df0a123a09e7ecc   2 hours ago         Running             sidecar             0
824f26368efc0       sha256:5feec37454f45d060c5f528c7d0bd4958df39e7ffd2e65ae42aae68bf78f69a5   2 hours ago         Running             dnsmasq             0
f3d35b783af1e       sha256:5d049a8c4eec92b21ca4be399c260166d96569a1a52d497f4a0365bb55c1a18c   2 hours ago         Running             kubedns             0
```
