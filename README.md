# AWSで構築するKubernetes The Hard Way

## **はじめに**
---

<br>

Kubernetes The Hard Wayは、Kubernetesのクラスタを1から手動で作成する手順を記載しています。

そのため、完全に自動化されて管理されたKubernetesクラスタを作成する手順を探している方のものではありません。

本手順は[Kelsey Hightower](https://github.com/kelseyhightower)によって執筆された[オリジナルのチュートリアル](https://github.com/kelseyhightower/kubernetes-the-hard-way)を参考に作成しています。

また、オリジナルのチュートリアルは[GCP(Google Cloud Platform)](https://cloud.google.com/docs/overview?hl=ja)を基盤とした手順であり、本ドキュメントとは環境差分があります。

<br>

## **想定読者**
---
以下を想定しています。
- Kubernetesクラスタの学習を始めたばかりで、全てのコンポーネントがどのように組み合わさっているかを理解したい人

- Kubernetesの概要は知っているけど、実施に手を動かしたことがない人

<br>

## **今回出来上がるクラスタの詳細**
---

<br>

Kubernetes The Hard Wayでは、コンポーネント間のエンドツーエンドの暗号化とRBAC認証ができる、可用性の高いKubernetesクラスタを構築します。

使用するコンポーネントとそのバージョン一覧は下記の通りです。

- Kubernetes v1.21.0
- containerd v1.14.4
- coredns v1.18.3
- cni v0.9.1
- etcd v3.4.15

<br>

## **目次**
---

<br>

<nav>　
    <ul>　　
        <li><a href=https://github.com/takei-ryuichi/Kubernetes-The-Hard-Way-AWS-/blob/main/docs/01-prerequisites.md target=”_blank”>01-前提条件</a>
        <li><a href=https://github.com/takei-ryuichi/Kubernetes-The-Hard-Way-AWS-/blob/main/docs/02-client-tools.md target=”_blank”>02-クライアントツールのインストール</a>
        <li><a href=https://github.com/takei-ryuichi/Kubernetes-The-Hard-Way-AWS-/blob/main/docs/03-compute-resource.md target=”_blank”>03-EC2でComputing Resourceをプロビジョニング</a>
        <li><a href= target=”_blank”>04-認証局(CA)のプロビジョニングとTLS証明書の生成</a>
        <li><a href= target=”_blank”>05-認証用kubeconfigの生成</a>
        <li><a href= target=”_blank”>06-暗号化の設定とキーの生成</a>
        <li><a href= target=”_blank”>07-etcdの起動</a>
        <li><a href= target=”_blank”>08-Kubernetesコントロールプレーンの起動</a>
        <li><a href= target=”_blank”>09-ワーカーノードの起動</a>
        <li><a href= target=”_blank”>10-リモートアクセス用のkubectl設定</a>
        <li><a href= target=”_blank”>11-クラスタ内ネットワークの設定</a>
        <li><a href= target=”_blank”>12-DNSクラスタアドオンの導入</a>
        <li><a href= target=”_blank”>13-スモークテスト</a>
        <li><a href= target=”_blank”>14-crictlを使用してワーカーノードのイメージ・ポッド・コンテナをチェックする</a>
        <li><a href=15-後片付け.html target=”_blank”>15-後片付け</a>
    </ul>
</nav>
