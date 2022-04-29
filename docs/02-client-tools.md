
## **02-クライアントツールのインストール**

<br>

この手順では`cfssl`, `cfssljson`, `kubectl`のインストールを行います。

<br>

### **CFSSLとCFSSLJSONのインストール**
---

<br>

`cvfssl`と`cfssljson`はPKI環境の構築とTLSの証明書発行に使用するツールです。

<br>

#### **OS Xでのインストール**
---

<br>

OS X(Mac OS)での手順を以下に示します。
```
curl -o cfssl https://pkg.cfssl.org/R1.2/cfssl_darwin-amd64
curl -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_darwin-amd64
```

```
chmod +x cfssl cfssljson
```

```
sudo mv cfssl cfssljson /usr/local/bin/
```

OS Xを使用し、pre-buildでインストールしようとした場合、問題が生じる場合があります。

その場合はHomebrewを使用してインストールします。

```
brew install cfssl
```

<br>

#### **Linuxでのインストール**
---

<br>

Linux環境でのインストール手順を以下に示します。

```
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

<br>

#### **確認**
---

<br>

cfsslのバージョンを確認します。

```
cfssl version
```

動作確認済みのバージョンは以下です。

```
Version: 1.6.1
Runtime: go1.17.2
```

(cfssljsonにはバージョンを表示するコマンドがありません）

<br>

#### **kubectlのインストール**
---

<br>

`kubectl`はKubernetes API Serverとの通信に使用されるコマンドラインツールです。
今回は`v1.22.0`を使用しています。

<br>

##### **OS X**
---

<br>

```
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/darwin/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

<br>

#### **Linux**
---

<br>

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/linux/amd64/kubectl
```
```
chmod +x kubectl
```
```
sudo mv kubectl /usr/local/bin/
```

<br>

#### **確認**
---

<br>

kubectlバージョン1.22.0がインストールされていることを確認します。

```
kubectl version --client
```

> 出力例

<br>

```
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.0", GitCommit:"c2b5237ccd9c0f1d600d3072634ca66cefdf272f", GitTreeState:"clean", BuildDate:"2021-08-04T18:03:20Z", GoVersion:"go1.16.6", Compiler:"gc", Platform:"darwin/arm64"}
```
