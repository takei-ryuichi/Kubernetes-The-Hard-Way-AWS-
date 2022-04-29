
## **10-リモートアクセス用のkubectl設定**
---

<br>

このセクションでは、adminユーザーのcredenialに基づいた、kubectlコマンドラインユーティリティ用のkubeconfigファイルを生成します。

adminのクライアント証明書の生成に使用したディレクトリと同じディレクトリでコマンドを実行してください。

<br>

### **Admin Kubernetes設定ファイル**
---

<br>

各kubeconfigは、Kubernetes APIサーバーと接続できる必要があります。

高可用性を実現するために、Kubernetes APIサーバーの前に設置した外部ロードバランサーに割り当てられたIPアドレスを使用します。

adminユーザーを認証する`kubeconfig`ファイルを生成します。

```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
--load-balancer-arns ${LOAD_BALANCER_ARN} \
--output text --query 'LoadBalancers[].DNSName')

kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem

kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin

kubectl config use-context kubernetes-the-hard-way
```

<br>

### **確認**
---

<br>

remoteのKubernetesクラスタのノードの一覧を取得します。

```
kubectl get nodes
```

> 出力例

```
NAME           STATUS   ROLES    AGE   VERSION
ip-10-0-1-20   Ready    <none>   89m   v1.21.0
ip-10-0-1-21   Ready    <none>   89m   v1.21.0
ip-10-0-1-22   Ready    <none>   89m   v1.21.0
```
