## **12-DNSクラスタアドオンの導入**
---

<br>

このセクションでは、Kubernetesクラスタ内で動いているアプリケーションに、CoreDNSを使ったDNSベースのサービスディスカバリを提供するDNSアドオンをデプロイします。

<br>

### **DNSクラスターアドオン**
---

<br>

corednsクラスターアドオンをデプロイします。

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.8.yaml
```

> 出力

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

`kube-dns deployment`によって作られたPodの確認を行います。

```
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> 出力例

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-8494f9c688-jth4j   1/1     Running   0          46s
coredns-8494f9c688-p679g   1/1     Running   0          46s
```

<br>

### **確認**
---

<br>

busybox deploymentを作成します。

```
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

busybox deploymentによって作られてPodを確認します。

```
kubectl get pods -l run=busybox
```

> 出力例

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          25s
```

busybox pod内からkubernetesserviceのDNS lookupを行います。

```
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
echo ${POD_NAME}

kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> 出力例

```
ubernetes
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```
