## **05-認証用kubeconfigの生成**
---

<br>

このセクションでは、Kubernetes APIサーバーがKubernetesのクライアントを配置、認証できるようにするためのkubeconfig(Kubernetes構成ファイル)を生成します。

<br>

### **クライアントの認証設定**
---

<br>

以下のファイルを生成します。
- conttoller-manager
- kubelet
- kube-proxy
- scheduler
- adminユーザー用のkubeconfig

<br>

### **KubernetesのPublic DNSアドレス**
---

<br>

各`kubeconfig`はKubernetes APIサーバと接続できることが要求されます。
高可用性を実現するために、Kubernetes APIサーバの前に設置されている外部ロードバランサーのIPアドレスを使用します。

`kubernetes-the-hard-way` のDNSアドレスを取得します。

```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[0].DNSName')
echo ${KUBERNETES_PUBLIC_ADDRESS}
```

<br>

### **kubeletのkubeconfigsの生成**

<br>

kubelet用のkubeconfigファイルを生成するときは、`kubelet`のノード名と同じクライアント証明書を使用する必要があります。

これにより、kubeletがKubernetesのNode Authorizerによって認可されるようになります。

ワーカーノード毎にkubeconfigを生成します。

```
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

以下のファイルが生成されます。

```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

<br>

### **kube-proxyのkubeconfigの生成**
---

<br>

kube-proxyのkubeconfigも生成します。

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

以下のファイルが生成されます。

```
kube-proxy.kubeconfig
```

<br>

### **kube-controller-managerのkubeconfigの生成**
---

<br>

kube-controller-managerのkubeconfigを生成します。

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

以下のファイルが生成されます。

```
kube-controller-manager.kubeconfig
```

<br>

### **kube-schedulerのkubeconfig**
---

<br>

kube-schedulerのkubeconfigを生成します。

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

以下のファイルが生成されます。

```
kube-scheduler.kubeconfig
```

<br>

### **adminユーザー用のkubeconfigの生成**
---

<br>

adminユーザーのkubeconfigを生成します。


```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

以下のファイルが生成されます。

```
admin.kubeconfig
```

<br>

### **kubeconfigの配布**
---

<br>

kubeletとkube-proxyのkubecnofigをコピーし、各ワーカーノードに配置します

```
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa \
    ${instance}.kubeconfig kube-proxy.kubeconfig ubuntu@${external_ip}:~/
done
```
> 出力例

```
worker-0.kubeconfig                           100% 6449   260.8KB/s   00:00    
kube-proxy.kubeconfig                         100% 6371   261.1KB/s   00:00    
worker-1.kubeconfig                           100% 6449   273.1KB/s   00:00    
kube-proxy.kubeconfig                         100% 6371   268.2KB/s   00:00    
worker-2.kubeconfig                           100% 6449   265.6KB/s   00:00    
kube-proxy.kubeconfig                         100% 6371   287.2KB/s   00:00 
```

kube-controller-managerとkube-schedulerのkubeconfigをコピーし、各コントローラーノードに配置します。

```
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa \
    admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ubuntu@${external_ip}:~/
done
```

> 出力例

```
admin.kubeconfig                              100% 6265   255.2KB/s   00:00    
kube-controller-manager.kubeconfig            100% 6387    51.6KB/s   00:00    
kube-scheduler.kubeconfig                     100% 6241   249.1KB/s   00:00    
admin.kubeconfig                              100% 6265   276.1KB/s   00:00    
kube-controller-manager.kubeconfig            100% 6387   271.4KB/s   00:00    
kube-scheduler.kubeconfig                     100% 6241   122.2KB/s   00:00    
admin.kubeconfig                              100% 6265   278.5KB/s   00:00    
kube-controller-manager.kubeconfig            100% 6387   127.7KB/s   00:00    
kube-scheduler.kubeconfig                     100% 6241   276.7KB/s   00:00
```