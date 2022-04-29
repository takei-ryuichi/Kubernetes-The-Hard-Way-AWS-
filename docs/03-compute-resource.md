## **03-EC2でComputing Resourceをプロビジョニング**

---

Kubernetesには、Kubernetesコントロールプレーンと、コンテナが最終的に実行されるワーカーノードをホストするための一連のマシンが必要です。
このセクションでは、単一のコンピューティングゾーンで安全で可用性の高いKubernetesクラスターを実行するために必要なコンピューティングリソースをプロビジョニングします。

<br>

### **Networking**
---

<br>

[Kubernetesのネットワークモデル](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model)は、コンテナとノードが相互に通信できるフラットネットワークを想定しています。
これが正しく構成されていない場合、[ネットワークポリシー](https://kubernetes.io/docs/concepts/services-networking/network-policies/)により、コンテナのグループが相互に通信したり、外部ネットワークエンドポイントと通信したりする方法が制限される可能性があります。
> Network Policyの設定は、このチュートリアルの範囲外です。

<br>

#### **仮想プライベートクラウドネットワーク(VPC)**
---

<br>

このステップでは、Kubernetesクラスタをホストするために専用の[仮想プライベートクラウド(VPC)](https://kubernetes.io/docs/concepts/services-networking/network-policies/)やネットワークをセットアップします。
`kubernetes-the-hard-way`のValueを持ったVPCを作成します。

<br>

#### **VPC**
---

<br>

```
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --output text --query 'Vpc.VpcId')
aws ec2 create-tags --resources ${VPC_ID} --tags Key=Name,Value=kubernetes-the-hard-way
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-support '{"Value": true}'
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-hostnames '{"Value": true}'
```

<br>

#### **Subnet**
---

<br>

[サブネット](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/configure-subnets.html#subnet-basics)は、Kubernetesクラスタ内の各ノードにプライベートIPアドレスを割り当てるためのリソースです。

十分な大きさのIPレンジでプロビジョニングする必要があります。

以下は`kubernetes-the-hard-way`のVPCに`kubernetes`サブネットをプロビジョニングしています。

> チュートリアルでは、10.0.1.0/24で最大254のコンピューティングインスタンスをホストできます。

```
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.0.1.0/24 \
  --output text --query 'Subnet.SubnetId')
aws ec2 create-tags --resources ${SUBNET_ID} --tags Key=Name,Value=kubernetes
```

<br>

#### **Internet Gateway**
---

<br>

[インターネットゲートウェイ](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/VPC_Internet_Gateway.html)はKubernetesのコントローラが外部からAPIのエンドポイントとなるために必要な構成です。

以下は`kubernetes-the-hard-way`のVPCに`kubernetes`Internet Gatewayを作成しています。

```
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway --output text --query 'InternetGateway.InternetGatewayId')
aws ec2 create-tags --resources ${INTERNET_GATEWAY_ID} --tags Key=Name,Value=kubernetes
aws ec2 attach-internet-gateway --internet-gateway-id ${INTERNET_GATEWAY_ID} --vpc-id ${VPC_ID}
```

<br>

#### **Route Tables**
---

<br>

[ルートテーブル](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/VPC_Route_Tables.html)は、VPC内の仮想ルータが参照するルーティングテーブルです。

以下は`kubernetes-the-hard-way`のVPCに`0.0.0.0/0`のデフォルトルートを設定しています。

```
ROUTE_TABLE_ID=$(aws ec2 create-route-table --vpc-id ${VPC_ID} --output text --query 'RouteTable.RouteTableId')
aws ec2 create-tags --resources ${ROUTE_TABLE_ID} --tags Key=Name,Value=kubernetes
aws ec2 associate-route-table --route-table-id ${ROUTE_TABLE_ID} --subnet-id ${SUBNET_ID}
aws ec2 create-route --route-table-id ${ROUTE_TABLE_ID} --destination-cidr-block 0.0.0.0/0 --gateway-id ${INTERNET_GATEWAY_ID}
```

<br>

#### **Security Groups**
---

<br>

[セキュリティグループ](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/VPC_SecurityGroups.html)はVPC内の仮想ファイアウォールとして機能します。

以下はKubernetesクラスタ間の内部通信と外部SSH,ICMPおよびHTTPSを許可するセキュリティポリシーを記載しています。

```
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name kubernetes \
  --description "Kubernetes security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')
aws ec2 create-tags --resources ${SECURITY_GROUP_ID} --tags Key=Name,Value=kubernetes
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol all --cidr 10.0.0.0/16
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol all --cidr 10.200.0.0/16
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 6443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol icmp --port -1 --cidr 0.0.0.0/0
```

<br>

#### **Kubernetes Public Access**
---

<br>

Kubernetes APIサーバのエンドポイントとして割り当てるIPアドレスを設定します。

```
  LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
    --name kubernetes \
    --subnets ${SUBNET_ID} \
    --scheme internet-facing \
    --type network \
    --output text --query 'LoadBalancers[].LoadBalancerArn')
  TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
    --name kubernetes \
    --protocol TCP \
    --port 6443 \
    --vpc-id ${VPC_ID} \
    --target-type ip \
    --output text --query 'TargetGroups[].TargetGroupArn')
  aws elbv2 register-targets --target-group-arn ${TARGET_GROUP_ARN} --targets Id=10.0.1.1{0,1,2}
  aws elbv2 create-listener \
    --load-balancer-arn ${LOAD_BALANCER_ARN} \
    --protocol TCP \
    --port 443 \
    --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
    --output text --query 'Listeners[].ListenerArn'
  ```

  ```
  KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[].DNSName')
  ```

<br>

### **インスタンス**
---

<br>

このステップでは、KubernetesクラスタのホストとなるVMをプロビジョニングします。

<br>

#### **Instance Image**
---

<br>

利用するインスタンスイメージを定義します。

今回はパブリックとして公開しているubuntuの[AMI](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/AMIs.html)を指定しています。



```
IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
  --output json \
  --filters \
  'Name=root-device-type,Values=ebs' \
  'Name=architecture,Values=x86_64' \
  'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*' \
  | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')
```

jqコマンドが入っていない場合は以下のコマンドでインストールを行います。

```
brew install jq
```

<br>

#### **SSH Key Pair**
---

<br>

EC2インスタンスにSSH接続するためのKey Pairを作成します。

```
aws ec2 create-key-pair --key-name kubernetes --output text --query 'KeyMaterial' > kubernetes.id_rsa
chmod 600 kubernetes.id_rsa
```

<br>

#### **Kubernetes コントローラノード**
---

<br>

[Amazon EC2](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/concepts.html)で、Kubernetesコントローラノードのホストとなる仮想マシンを作成します。

今回は `t3.micro` インスタンスを使用し、3台の仮想マシンを立ち上げます。

```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name kubernetes \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t3.micro \
    --private-ip-address 10.0.1.1${i} \
    --user-data "name=controller-${i}" \
    --subnet-id ${SUBNET_ID} \
    --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${instance_id} --no-source-dest-check
  aws ec2 create-tags --resources ${instance_id} --tags "Key=Name,Value=controller-${i}"
  echo "controller-${i} created "
done
```

<br>

#### **Kubernetesワーカーノード**
---

<br>

[Amazon EC2](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/concepts.html)で、Kubernetesワーカーノードのホストとなる仮想マシンを作成します。

今回は `t3.micro` インスタンスを使用し、3台の仮想マシンを立ち上げます、

> 今回は`t3.micro`でプロビジョニングしていますが、ワーカーノードはコンテナアプリケーションが展開、実行される場所であるため
実際のサイジングは非常に重要です。
アプリケーションの要件に応じたリソース計算が求められます。

```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name kubernetes \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t3.micro \
    --private-ip-address 10.0.1.2${i} \
    --user-data "name=worker-${i}|pod-cidr=10.200.${i}.0/24" \
    --subnet-id ${SUBNET_ID} \
    --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${instance_id} --no-source-dest-check
  aws ec2 create-tags --resources ${instance_id} --tags "Key=Name,Value=worker-${i}"
  echo "worker-${i} created"
done
```
