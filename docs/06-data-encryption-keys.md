## **06-暗号化の設定とキーの生成**
---

<br>

Kubernetesは、クラスタの状態、アプリケーションの設定、秘匿情報などを含むさまざまなデータが格納されます。

そのデータを守るために、Kubernetesはクラスタ内で保持しているデータを暗号化する機能が提供されています。

このセクションでは、Kubernetes Secretsの暗号化に合わせた暗号化鍵と暗号化の設定ファイルを生成します。

<br>

### **暗号化鍵**
---

<br>

暗号化に使用する鍵を作成します。

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

<br>

### **暗号化の設定ファイル**
---

<br>

暗号化の設定のための`encryption-config.yaml`を生成します。

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

`encryption-config.yaml`をコピーし、各コントローラーノードに配置します。

```
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  
  scp -i kubernetes.id_rsa encryption-config.yaml ubuntu@${external_ip}:~/
done
```

> 出力例

```
encryption-config.yaml                        100%  240    10.4KB/s   00:00    
encryption-config.yaml                        100%  240     5.2KB/s   00:00    
encryption-config.yaml                        100%  240     4.9KB/s   00:00 
```

