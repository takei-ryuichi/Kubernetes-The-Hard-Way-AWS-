## **01-前提条件**

### **Amazon Web Service**
---
<br>
このチュートリアルではAmazon Web Service(以下AWS)を利用して、kubernetesクラスターを立ち上げます。
このチュートリアルを通して掛かる費用は、24時間あたり2ドル未満です。
また、今回使用するリソースはAWSの無料枠を超えるので、チュートリアル終了後には作成したリソースをクリーンアップし、不要なコストが発生しないよう注意してく
ださい。

<br>

### **Amazon Web Service CLI**

#### **AWS CLIのインストール**
---
<br>

[AWSの公式ドキュメント](https://aws.amazon.com/jp/cli/)に従って、AWS CLIをインストールし、必要な設定を行います。

インストール完了後、以下のコマンドでAWS CLIが有効であることを確認します。

```
aws --version
```

<br>

#### **デフォルトリージョンの設定**
---
<br>

このチュートリアルで使用するデフォルトリージョンを設定します。

```
AWS_REGION=ap-northeast-1
aws configure set default.region $AWS_REGION
```

<br>

### **tmuxを使ったパラレルなコマンド実行**
---

<br>

オリジナルのチュートリアルでも推奨の設定として記載されています。
この手順をスキップしてもKubernetesクラスタの動作に影響はありません。

<br>

https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/01-prerequisites.md#running-commands-in-parallel-with-tmux

<br>

### **作業用ディレクトリの作成**
---

<br>

本チュートリアルでは、多くのファイル生成や転送の処理を行います。
また、リモート接続のためのキーペアの払い出しも行います。
ファイル管理の観点から、予め作業用ディレクトリを作成し、そこをチュートリアル中のカレントディレクトリとすることを推奨します。
