## **11-クラスタ内ネットワークの設定**
---

<br>

ノードにスケジュールされたPodは、ノードのPod CIDR範囲からIPアドレスを受け取ります。

この時点では、ネットワークルートが見つからないため、Podは異なるノードで実行されている他のPodと通信できません。

このセクションでは、ノードのPod CIDR範囲をノードの内部IPアドレスにマップするための、各ワーカーノードのルートを作成します。

Kubernetesのネットワークモデルの実装は他にもあります。

<br>

### **ルーティングテーブルとルート群**
---

<br>

このセクションでは、kubernetes-the-hard-wayVPCネットワーク内でルートを作るために必要な情報を集めます。

通常、この機能は`flanne`, `calico`, `amazon-vpc-cin-k8s`等のCNIプラグインによって提供されます。
これを手作業で行うことにより、これらのプラグインがバックグラウンドで何をしているのかを理解しやすくなります。

まずは各ワーカーインスタンスの内部IPアドレスとPod CIDR範囲を表示させます。

```
for instance in worker-0 worker-1 worker-2; do
  instance_id_ip="$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress]')"
  instance_id="$(echo "${instance_id_ip}" | cut -f1)"
  instance_ip="$(echo "${instance_id_ip}" | cut -f2)"
  pod_cidr="$(aws ec2 describe-instance-attribute \
    --instance-id "${instance_id}" \
    --attribute userData \
    --output text --query 'UserData.Value' \
    | base64 --decode | tr "|" "\n" | grep "^pod-cidr" | cut -d'=' -f2)"
  echo "${instance_ip} ${pod_cidr}"

  aws ec2 create-route \
    --route-table-id "${ROUTE_TABLE_ID}" \
    --destination-cidr-block "${pod_cidr}" \
    --instance-id "${instance_id}"
done
```

> 出力例

```
10.0.1.20 10.200.0.0/24
{
    "Return": true
}
10.0.1.21 10.200.1.0/24
{
    "Return": true
}
10.0.1.22 10.200.2.0/24
{
    "Return": true
}
```

<br>

### **ルートの確認**
---

<br>

各ワーカーインスタンスに対してネットワークルートを確認します。

```
aws ec2 describe-route-tables \
  --route-table-ids "${ROUTE_TABLE_ID}" \
  --query 'RouteTables[].Routes'
```

> 出力例

```
[
    [
        {
            "DestinationCidrBlock": "10.200.0.0/24",
            "InstanceId": "i-0cb0e48788838daa4",
            "InstanceOwnerId": "523358537305",
            "NetworkInterfaceId": "eni-0d5ff998bd2fb09c5",
            "Origin": "CreateRoute",
            "State": "active"
        },
        {
            "DestinationCidrBlock": "10.200.1.0/24",
            "InstanceId": "i-001c9deec822b1325",
            "InstanceOwnerId": "523358537305",
            "NetworkInterfaceId": "eni-04334cbdcbcf2cfd5",
            "Origin": "CreateRoute",
            "State": "active"
        },
        {
            "DestinationCidrBlock": "10.200.2.0/24",
            "InstanceId": "i-0055e9af229f7ea5d",
            "InstanceOwnerId": "523358537305",
            "NetworkInterfaceId": "eni-0c28e31352ccd881a",
            "Origin": "CreateRoute",
            "State": "active"
        },
        {
            "DestinationCidrBlock": "10.0.0.0/16",
            "GatewayId": "local",
            "Origin": "CreateRouteTable",
            "State": "active"
        },
        {
            "DestinationCidrBlock": "0.0.0.0/0",
            "GatewayId": "igw-0f1932111bd00691f",
            "Origin": "CreateRoute",
            "State": "active"
        }
    ]
]
```
