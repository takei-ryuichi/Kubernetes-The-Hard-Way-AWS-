# AWSで構築するKubernetes The Hard Way

### **はじめに**
---

<br>

Kubernetes The Hard Wayは、Kubernetesのクラスタを1から手動で作成する手順を記載しています。

そのため、完全に自動化されて管理されたKubernetesクラスタを作成する手順を探している方のものではありません。

本手順は[Kelsey Hightower](https://github.com/kelseyhightower)によって執筆された[オリジナルのチュートリアル](https://github.com/kelseyhightower/kubernetes-the-hard-way)を参考に作成しています。

また、オリジナルのチュートリアルは[GCP(Google Cloud Platform)](https://cloud.google.com/docs/overview?hl=ja)を基盤とした手順であり、本ドキュメントとは環境差分があります。

<br>

### **想定読者**
---
以下を想定しています。
- Kubernetesクラスタの学習を始めたばかりで、全てのコンポーネントがどのように組み合わさっているかを理解したい人

- Kubernetesの概要は知っているけど、実施に手を動かしたことがない人

<br>

### **今回出来上がるクラスタの詳細**
---

<br>

Kubernetes The Hard Wayでは、コンポーネント間のエンドツーエンドの暗号化とRBAC認証ができる、可用性の高いKubernetesクラスタを構築します。

使用するコンポーネントとそのバージョン一覧は下記の通りです。

- Kubernetes v1.21.0
- containerd v1.14.4
- coredns v1.18.3
- cni v0.9.1
- etcd v3.4.15
