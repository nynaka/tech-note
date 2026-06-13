---
title: KCNA-JP 試験概要
description: KCNA-JP（Kubernetes and Cloud Native Associate）は、Linux FoundationとCNCFが提供するKubernetesおよびクラウドネイティブエコシステムに関するエントリーレベルの日本語認定試験です。選択式60問・90分・合格ライン75%の構成で、Kubernetes基礎からクラウドネイティブアーキテクチャまでの幅広い知識を問います。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

# KCNA-JP (Kubernetes and Cloud Native Associate) 試験

## 1. 試験概要

KCNA-JPは、Linux FoundationとCNCFが提供する、Kubernetesおよびクラウドネイティブエコシステムに関するエントリーレベルの認定試験（日本語版）です。英語版のKCNAとは別に申し込む必要があります。試験は完全オンラインのリモート監視（プロクター）形式で実施されます。

* **試験形式**: 選択式（多肢選択問題） ※実技試験（コマンド操作）はありません
* **問題数**: 60問
* **試験時間**: 90分
* **合格ライン**: 75%以上（45問正解）
* **受験料**: 約40,000円（受験料には再受験（リテイク）1回分が含まれており、計2回受験できます）
* **有効期限**: 2年間
* **前提条件**: なし（初心者でも受験可）
* **受験環境**: 自宅等からオンライン受験。Webカメラ・マイク・安定したインターネット接続が必要。身分証明書（パスポート、運転免許証、マイナンバーカード等）も必要。

## 2. 試験範囲

試験範囲は4つの分野に分かれており、Kubernetesの基礎だけでなく、周辺のクラウドネイティブ技術（CNCF Landscapeに属するツール群など）も広く含まれます。公式カリキュラムは [CNCF GitHub](https://github.com/cncf/curriculum) で公開されています。

1. **Kubernetes Fundamentals (44%)**
   * Kubernetesコアコンセプト（Pod、Deployment、Service、Namespace等）、管理、スケジューリング、コンテナ化。
2. **コンテナオーケストレーション (28%)**
   * ネットワーク、セキュリティ、トラブルシューティング、ストレージ。
3. **クラウドネイティブアプリケーションの配信 (16%)**
   * アプリケーション配信（CI/CD、GitOps等）、デバッグ。
4. **クラウドネイティブアーキテクチャ (12%)**
   * 可観測性（Prometheus、テレメトリ等）、クラウドネイティブエコシステムと原則、クラウドネイティブコミュニティとコラボレーション。

> **注意**: 過去の資料では5分野（Kubernetes基礎 46%、コンテナオーケストレーション 22%、クラウドネイティブアーキテクチャ 16%、クラウドネイティブの可観測性 8%、クラウドネイティブアプリケーションの配信 8%）と記載されているものがありますが、現在は上記の4分野構成に改訂されています。受験前に [公式カリキュラム](https://github.com/cncf/curriculum) で最新情報を確認してください。

## 3. 日本語での勉強方法・参考サイト

### 勉強方法

実技試験がなく概念や知識を問う問題が中心のため、基礎の理解と問題演習の反復が合格への近道です（目安学習時間：20〜30時間程度）。

1. **基礎知識の習得**
   * まずは書籍等でKubernetesの基本概念（Pod、Deployment、Service等のリソースやアーキテクチャ）を理解します。
2. **模擬試験の反復（最重要）**
   * Udemy等で提供されている問題集を活用し、「初見で解く → 解説を読み込む → 分からない用語を調べる → 9割以上取れるまで繰り返す」というサイクルを回すのが多くの合格者が実践している定番の学習法です。
3. **周辺エコシステムの理解**
   * Kubernetes本体だけでなく、Prometheus、Fluentd、Linkerd、ArgoCDなどの周辺ツールの概要を知っておく必要があります。CNCF Landscapeを活用し、各ツールがどのカテゴリ（可観測性、サービスメッシュ、GitOpsなど）に該当するかを整理しておきましょう。
4. **英語用語への慣れ**
   * 試験は日本語ですが、一部の専門用語は英語のまま、あるいは直訳に近い表現で出題されることがあるため、英語のオリジナル用語にも馴染んでおくと安心です。

### ドメイン別 重点ポイント

| ドメイン                                 | 配点    | 重点的に押さえるトピック                                                                                                                                                                       |
| ---------------------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Kubernetes Fundamentals                  | **44%** | Podのライフサイクル、Deployment/ReplicaSet、Service（ClusterIP/NodePort/LoadBalancer）、Namespace、ConfigMap/Secret、コントロールプレーンのコンポーネント（kube-apiserver, etcd, scheduler等） |
| コンテナオーケストレーション             | **28%** | CNI（ネットワークプラグイン）、NetworkPolicy、RBAC・PodSecurity、PV/PVC、コンテナランタイム（containerd等）                                                                                    |
| クラウドネイティブアプリケーションの配信 | **16%** | CI/CDパイプライン、GitOps（ArgoCD、Flux）、Helm、イメージビルドのベストプラクティス                                                                                                            |
| クラウドネイティブアーキテクチャ         | **12%** | Prometheus・Grafanaによる監視、分散トレーシング（Jaeger等）、ログ集約（Fluentd等）、マイクロサービスとサービスメッシュ（Istio、Linkerd）、CNCFガバナンス                                       |

### おすすめ学習スケジュール（3〜4週間プラン）

* **1〜2週目**: Kubernetes基礎（Fundamentals）を重点的に学習。公式ドキュメントや書籍でコアコンセプトを体系的に理解する。
* **3週目**: コンテナオーケストレーション・アプリケーション配信・クラウドネイティブアーキテクチャを横断的に学習。CNCF Landscapeでエコシステムを俯瞰する。
* **4週目**: Udemyの模擬試験を繰り返し解く。正答率9割以上を目標に、間違えた問題の関連概念を都度深掘りする。

### 参考サイト・教材

* **公式カリキュラム（CNCF GitHub）**
  * [https://github.com/cncf/curriculum](https://github.com/cncf/curriculum) ─ 試験範囲の正式な定義。最新のドメイン・割合はここで確認する。
* **Kubernetes公式ドキュメント**
  * [https://kubernetes.io/ja/docs/](https://kubernetes.io/ja/docs/) ─ 日本語版あり。Pod、Deployment、Service等の基本概念はここで体系的に学べる。
* **CNCF Landscape**
  * [https://landscape.cncf.io/](https://landscape.cncf.io/) ─ クラウドネイティブエコシステムのツール群を一覧できる。各カテゴリ（可観測性、サービスメッシュ、GitOps等）との対応を把握するのに不可欠。
* **Udemy**
  * 「これだけでOK！【KCNA-JP】認定Kubernetesクラウドネイティブアソシエイト模擬試験問題集」などの模擬試験が非常に定評があります。
* **書籍**
  * 『Kubernetes実践ガイド』（インプレス）などのKubernetes入門書。
* **公式トレーニング**
  * [LFS250-JP（Kubernetesとクラウドネイティブ基礎）](https://training.linuxfoundation.org/ja/training/kubernetes-and-cloud-native-essentials-lfs250-jp/) ─ 有料の公式オンラインコース。「試験 + LFS250-JP」のバンドル購入で割引あり（単体より安くなる場合がある）。
* **各種技術ブログ（Qiita, Zennなど）**
  * 「KCNA-JP 合格」等で検索すると、多くの受験者の合格体験記や詳細な勉強方法、まとめノートが見つかるため、学習計画の参考になります。

## 4. 合格後のキャリアパス

KCNA-JPはエントリーレベルの認定資格であり、合格後は実践的なスキルを証明する上位資格へのステップアップが推奨されています。

```
KCNA-JP（概念・知識）
    ↓
CKA-JP（Certified Kubernetes Administrator）─ クラスタ管理・運用の実技
    ↓
CKAD-JP（Certified Kubernetes Application Developer）─ アプリ開発・デプロイの実技
    ↓
CKS-JP（Certified Kubernetes Security Specialist）─ セキュリティ専門（CKA取得が前提）
```

* **資格の有効期限と更新**: 有効期限は2年間。更新には再受験して合格する必要があります（CARE プログラムにより、上位資格合格で自動更新される場合もあります）。
* **受験申込**: [Linux Foundation 公式サイト（日本語）](https://training.linuxfoundation.org/ja/certification/kubernetes-and-cloud-native-associate-kcna-jp/) から申し込めます。
