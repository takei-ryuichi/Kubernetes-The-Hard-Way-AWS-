## **13-スモークテスト**
---

<br>

このセクションでは、Kubernetesクラスタが正しく機能していることを確認するためのタスクを実行します。

<br>

### **データの暗号化**
---

<br>

このステップでは保存されているデータの暗号化を確認します。

generic secretを作ります。

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

etcdに保存されている`kubernetes-the-hard-way`secretをhexdumpします。

```
external_ip=$(aws ec2 describe-instances --filters \
  "Name=tag:Name,Values=controller-0" \
  "Name=instance-state-name,Values=running" \
  --output text --query 'Reservations[].Instances[].PublicIpAddress')

ssh -i kubernetes.id_rsa ubuntu@${external_ip} \
 "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> 出力例

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 67 3f 76 23 d3 0f 9b  |:v1:key1:g?v#...|
00000050  c2 92 14 54 6e 7f 26 41  a6 27 e0 a7 d6 9e 3f 67  |...Tn.&A.'....?g|
00000060  07 88 36 c9 99 ac dd e5  5f 44 e5 f0 7e 45 9b 0a  |..6....._D..~E..|
00000070  04 ed 0c b8 77 0b a7 29  7c df 34 ec 4c 22 d6 36  |....w..)|.4.L".6|
00000080  f7 58 38 b9 5f 49 1f 0f  b8 ac a6 ea 4d 23 95 0f  |.X8._I......M#..|
00000090  aa 35 c8 39 eb 33 e2 c8  4c 70 5e f8 2c 05 ef 88  |.5.9.3..Lp^.,...|
000000a0  cc 41 3f da d2 05 93 3a  3c 4d 1c 33 a2 fe 78 fb  |.A?....:<M.3..x.|
000000b0  ec fa 02 af cd c0 6d 8e  dd 6d b7 5a e2 b1 f7 44  |......m..m.Z...D|
000000c0  3c ec d9 04 7d 9b 82 5e  d4 22 fe 6f 5e 2b 47 aa  |<...}..^.".o^+G.|
000000d0  56 76 13 a0 9c a4 ca a6  c1 46 a1 5e 1b a6 ab 9b  |Vv.......F.^....|
000000e0  d8 71 e7 84 3c ed 94 a0  f6 b8 6e 11 2e 44 8e ab  |.q..<.....n..D..|
000000f0  0f f4 89 9a ac e6 cb f6  8f 48 da 8e 0e c2 ba cf  |.........H......|
00000100  c5 be 3f a4 c2 a0 38 29  78 23 a7 56 db b3 e0 20  |..?...8)x#.V... |
00000110  a3 ae d2 9b d7 8a 4b 3b  83 df ee 12 c5 71 1f e5  |......K;.....q..|
00000120  c6 5b 97 0a 98 02 9e 85  df db e2 70 44 37 35 b2  |.[.........pD75.|
00000130  a8 30 cf 79 b5 25 4b d3  7a 35 f6 cf 69 11 25 f2  |.0.y.%K.z5..i.%.|
00000140  bd 37 9e 2c 57 ed c0 d0  26 e0 8d b7 da bb 5e 76  |.7.,W...&.....^v|
00000150  0b e8 46 6d 6e 38 65 09  c2 0a                    |..Fmn8e...|
0000015a
```

etcdキーは、`k8s:enc:aescbc:v1:key1`というプレフィックスになっているはずです。

これは、aescbcプロバイダがkey1という暗号化キーでデータを暗号化したことを表しています。

<br>

### **自端末からDeploymentの作成と管理**
---

<br>

このステップではDeploymentの作成と管理ができているかを確認します。

nginx web serverのdeploymentを作成します。

```
kubectl create deployment nginx --image=nginx
```

nginx deploymentによってできたPodを確認します。

```
kubectl get pods -l app=nginx
```

> 出力例

```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6799fc88d8-qfjkw   1/1     Running   0          13s
```

<br>

### **Port Forwarding**
---

<br>

このステップでは、port forwardingを使って外部からアプリケーションにアクセスできるかを確認します。

nginx podのフルネームを取得します。

```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

ローカルの8080ポートをnginx Podの80番ポートにフォワードします。

```
kubectl port-forward $POD_NAME 8080:80
```

> 出力例

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

別のターミナルからフォワードしたアドレスにHTTPリクエストを投げてみます。

```
curl --head http://127.0.0.1:8080
```

> 出力例

```
HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Mon, 25 Apr 2022 05:07:07 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes
```

元のターミナルに戻ってnginx Podへのフォワーディングプロセスを止めます。

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

<br>

### **Logs**
---

<br>

このステップでは、コンテナのログの取得ができるかを確認します。

nginx Podのログを表示します。

```
kubectl logs $POD_NAME
```

> 出力例

```
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2022/04/25 05:05:55 [notice] 1#1: using the "epoll" event method
2022/04/25 05:05:55 [notice] 1#1: nginx/1.21.6
2022/04/25 05:05:55 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6) 
2022/04/25 05:05:55 [notice] 1#1: OS: Linux 5.13.0-1022-aws
2022/04/25 05:05:55 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/04/25 05:05:55 [notice] 1#1: start worker processes
2022/04/25 05:05:55 [notice] 1#1: start worker process 32
2022/04/25 05:05:55 [notice] 1#1: start worker process 33
127.0.0.1 - - [25/Apr/2022:05:07:07 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.79.1" "-"
```

<br>

### **Exec**
---

<br>

このステップではコンテナ内でのコマンド実行ができるかを確認します。

nginxコンテナに入って、nginx -vを実行してnginxのバージョンを表示します。

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> 出力例

```
nginx version: nginx/1.21.6
```

<br>

### **Services**
---

<br>

このステップでは、Serviceを使ったアプリケーションの公開ができるかを確認します。

nginx deploymentをNodePort を使って公開します。

```
kubectl expose deployment nginx --port 80 --type NodePort
```

クラスターがクラウドプロバイダーインテグレーションの設定がされていないため、LoadBalancerは使用できません。

この手順では、クラウドプロバイダーインテグレーションの設定は対象外です。

nginx serviceでアサインされたノードのポート取得します。

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

nginxノードポートへのリモートアクセスを許可するファイアウォールのルール追加します。

```
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port ${NODE_PORT} \
  --cidr 0.0.0.0/0
```
`nginx`podが実行されているワーカーノード名を取得します。

```
INSTANCE_NAME=$(kubectl get pod $POD_NAME --output=jsonpath='{.spec.nodeName}')
```

ワーカーインスタンスのExternal IPアドレスを取得します。

```
EXTERNAL_IP=$(aws ec2 describe-instances --filters \
    "Name=instance-state-name,Values=running" \
    "Name=network-interface.private-dns-name,Values=${INSTANCE_NAME}.*.internal*" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
```


External IPと`nginx`のノードポートを使用してHTTPリクエストを送信します。

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> 出力例

```
HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Mon, 25 Apr 2022 05:12:16 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes
```
