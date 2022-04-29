## **15-後片付け**
---

<br>

このセクションでは、これまでに作成してきたAWSに関するリソースを削除します。

<br>

### **EC2インスタンス**
---

<br>

コントローラーノード、ワーカーノードを削除します。

```
aws ec2 terminate-instances \
  --instance-ids \
    $(aws ec2 describe-instances \
      --filter "Name=tag:Name,Values=controller-0,controller-1,controller-2,worker-0,worker-1,worker-2" \
      --output text --query 'Reservations[].Instances[].InstanceId')
aws ec2 delete-key-pair --key-name kubernetes
```

<br>

### **Networking**

<br>

外部ロードバランサー、VPC等のネットワークリソースを削除します。

```
aws elbv2 delete-load-balancer --load-balancer-arn "${LOAD_BALANCER_ARN}"
```

```
aws elbv2 delete-target-group --target-group-arn "${TARGET_GROUP_ARN}"
```

```
aws ec2 delete-security-group --group-id "${SECURITY_GROUP_ID}"
ROUTE_TABLE_ASSOCIATION_ID="$(aws ec2 describe-route-tables \
  --route-table-ids "${ROUTE_TABLE_ID}" \
  --output text --query 'RouteTables[].Associations[].RouteTableAssociationId')"
```

```
aws ec2 disassociate-route-table --association-id "${ROUTE_TABLE_ASSOCIATION_ID}"
```

```
aws ec2 delete-route-table --route-table-id "${ROUTE_TABLE_ID}"
```

```
aws ec2 detach-internet-gateway \
  --internet-gateway-id "${INTERNET_GATEWAY_ID}" \
  --vpc-id "${VPC_ID}"
```

```
aws ec2 delete-internet-gateway --internet-gateway-id "${INTERNET_GATEWAY_ID}"
```

```
aws ec2 delete-subnet --subnet-id "${SUBNET_ID}"
```

```
aws ec2 delete-vpc --vpc-id "${VPC_ID}"
```
