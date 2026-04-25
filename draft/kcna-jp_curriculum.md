# KCNA-JP 受験対策：ハンズオン実技カリキュラム

KCNA-JPは知識や概念を問う問題が中心の試験ですが、実際に手を動かしてKubernetes環境を構築・操作することで、アーキテクチャや各コンポーネントの役割、周辺エコシステムとの連携についての理解が飛躍的に深まり、知識の定着に繋がります。

本カリキュラムは、素のLinux OS（DebianおよびFedora）へのKubernetesクラスタ構築から始まり、KCNA-JPの試験範囲に直結するクラウドネイティブ技術の基本操作までを網羅しています。

試験ドメインの配点比率は **Kubernetes基礎（46%）、コンテナオーケストレーション（22%）、クラウドネイティブアーキテクチャ（16%）、クラウドネイティブ可観測性（8%）、クラウドネイティブアプリケーションの配信（8%）** となっており、各フェーズはこの配点を意識した構成になっています。

---

## フェーズ1：Kubernetesクラスタの構築（環境構築）

クラスタをゼロから構築することで、Kubernetesのアーキテクチャ（Control PlaneとWorker Node）や、各コンポーネントの役割・相互関係を体感します。

:::info Control Planeコンポーネントの役割
クラスタを構成する主要コンポーネントと、それぞれの責務を整理します。

| コンポーネント | 役割 | 止まると？ |
|---|---|---|
| **kube-apiserver** | クラスタへの唯一の入口。全操作はここを経由する | `kubectl` が一切使えなくなる |
| **etcd** | クラスタの全状態を保存するKVストア | クラスタ設定が失われる（バックアップ必須） |
| **kube-scheduler** | 新しいPodをどのNodeに配置するかを決定する | 新規Podがスケジュールされなくなる |
| **kube-controller-manager** | ReconciliationループでDesired Stateを維持する | レプリカ数の維持やノード障害検知が止まる |
| **kubelet** | 各NodeでPodを起動・監視するエージェント（Worker Node常駐） | そのNode上のコンテナが管理できなくなる |
| **kube-proxy** | ServiceのIPルーティングをNode上で実現する | Service経由のネットワーク通信が止まる |

Kubernetesの設計思想である **「宣言的管理（Declarative Management）」** は、ユーザーが「あるべき状態（Desired State）」をマニフェストで宣言し、controller-managerのReconciliationループが現状（Actual State）とのズレを常に検知・修正することで実現されています。
:::

### 1-1. OSの初期設定と前提条件の構成（Debian / Fedora共通）
Kubernetesを動作させるためのOSレベルの準備を行います。
* **スワップの無効化** (`swapoff -a` および `/etc/fstab` の編集)  
  kubeletはノードのメモリリソースをCPUと同様に確定的に管理するため、スワップが有効だとメモリ使用量の予測が困難になります。そのためKubernetesはデフォルトでスワップが有効なノードへのデプロイを拒否します。
* **カーネルモジュールのロード** (`overlay`, `br_netfilter`)  
  `overlay` はコンテナのレイヤードファイルシステム（OverlayFS）に、`br_netfilter` はブリッジネットワーク経由のパケットをiptablesでフィルタするために必要です。
* IPv4フォワーディングの有効化とiptablesの設定 (`sysctl` の設定)
* SELinuxの無効化またはPermissiveモードへの変更（**Fedora環境**）
* ファイアウォール（`firewalld`/`ufw`）の適切なポート開放、または検証用の無効化

### 1-2. コンテナランタイムのインストール
KubernetesはコンテナをCRI（Container Runtime Interface）という標準インターフェースを通じて操作します。この仕組みにより、`containerd`や`CRI-O`など異なるランタイムを差し替えられます。
* `containerd` のインストールと設定作成
  * **Debian**: `apt` を用いたDocker公式GPGキー・リポジトリの追加とインストール
  * **Fedora**: `dnf` を用いたDocker公式リポジトリの追加とインストール
* `containerd` の設定ファイル (`config.toml`) で `SystemdCgroup = true` を有効化し再起動  
  kubeletとcontainerdのcgroupドライバーを `systemd` で統一することで、リソース管理の競合を防ぎます。

### 1-3. Kubernetesコンポーネントのインストール
クラスタ構築・管理に必要なツール群をインストールします。
* `kubeadm`, `kubelet`, `kubectl` のインストール
  * **Debian**: Kubernetes公式aptリポジトリの追加とインストール
  * **Fedora**: Kubernetes公式yumリポジトリの追加とインストール
* `kubelet` サービスの自動起動有効化 (`systemctl enable kubelet`)

### 1-4. クラスタの初期化とネットワーク設定
* `kubeadm init` コマンドによるControl Plane（マスターノード）の初期化
* 管理用ユーザーへの `kubectl` 認証設定（`~/.kube/config` のコピー）
* **CNI（Container Network Interface）プラグインのインストール**  
  CNIはPod間ネットワークの実装を担う標準インターフェースです。CNIプラグインを適用するまでPodはネットワーク設定が行われず`Pending`状態のままになります。CalicoやFlannelなどのマニフェストを `kubectl apply -f` で適用し、Pod間通信を確立します。  
  ※ CRI（コンテナ実行）・CNI（Pod間ネットワーク）・CSI（ストレージ）はそれぞれ独立した標準インターフェースであり、Kubernetesの拡張性を支える仕組みです。
* Worker Nodeのクラスタ参加（`kubeadm join`）と、ノードのステータス確認（`kubectl get nodes` で `Ready` になることの確認）

---

## フェーズ2：Kubernetesの基本リソース操作

「Kubernetes基礎（46%）」および「コンテナオーケストレーション（22%）」の試験範囲に対応します。配点が最も高いフェーズです。

### 2-1. Podとコンテナの基本操作
最小デプロイ単位であるPodのライフサイクルと、本番運用で重要な設定を学びます。

* `kubectl run` を用いたPodの単体起動
* YAMLマニフェストを用いたPodの宣言的作成 (`kubectl apply -f`)
* コンテナのログ確認 (`kubectl logs`) とコンテナ内でのコマンド実行 (`kubectl exec`)
* Podの状態推移（Pending, ContainerCreating, Running, CrashLoopBackOff等）の観察と、`kubectl describe pod` によるイベント確認
* **リソース要求と制限 (`requests` / `limits`)**  
  `requests` はスケジューラがPodを配置するNodeを決定する際に使う「最低保証量」、`limits` はコンテナが使用できる「上限量」です。`limits`超過時はOOMKillが発生します。
* **ヘルスチェック（Probe）の設定**
  * **livenessProbe**: コンテナが正常動作しているかを確認。失敗するとコンテナを再起動する
  * **readinessProbe**: コンテナがトラフィックを受け入れられる状態かを確認。失敗するとServiceのエンドポイントから除外される
  * **startupProbe**: 起動に時間がかかるアプリのための起動確認。startupProbeが成功するまでliveness/readinessProbeは実行されない

### 2-2. ワークロードの管理（Deployment）
アプリケーションの可用性とスケーリングを管理する手法を学びます。

* **Deploymentの構造理解**: Deploymentは `ReplicaSet` を管理し、`ReplicaSet` がPodを管理するという3層構造です。Podを直接作成・管理するのはReplicaSetであり、Deploymentはそのバージョン管理を担います。
* Deploymentマニフェストの作成と、`labels`/`selector` によるDeployment・ReplicaSet・Pod間の関連付け確認
* 複数レプリカ（Pod）の展開と、Podが意図せず削除されたときに自動復旧する（Self-healing）動作の確認
* トラフィック増加を想定したスケーリング操作 (`kubectl scale`)
* コンテナイメージの更新によるローリングアップデートと、障害発生時のロールバック (`kubectl rollout undo`)

### 2-3. 用途別ワークロードリソース
Deploymentの他にも、Kubernetesには目的に応じた複数のワークロードリソースがあります。

* **StatefulSet**: Podに安定したネットワークID（hostname）と永続ストレージを提供します。Deploymentでは各Podが匿名ですが、StatefulSetではPodの名前が `pod-0`, `pod-1` のように順序付けられ、再作成後も同じ名前・同じPVCが紐付きます。MySQLやZooKeeperなどのステートフルなアプリケーションに使用します。
* **DaemonSet**: クラスタの**全てのNode**（または指定条件を満たすNode）に1つずつPodを配置します。新しいNodeが追加されると自動的にPodが起動し、Nodeが削除されると自動的に除去されます。ログ収集エージェント（Fluent Bit等）やノード監視エージェントに使用します。
* **Job**: 処理が**完了**すること（exit code 0）を目的とするワークロードです。Deploymentと異なり、処理が終わってもPodは再起動されません。バッチ処理やデータベースのマイグレーションに使用します。
* **CronJob**: Jobをcron形式のスケジュール（例: `0 3 * * *`）で定期実行します。スケジュールに基づき、実行のたびにJobリソースとPodが生成されます。

### 2-4. Namespaceによるリソース分離
Namespaceはクラスタを論理的に分割する仕組みです。チームや環境（dev/staging/prod）ごとにリソースを分離し、マルチテナント運用の基礎となります。

* Namespaceの作成と、`-n <namespace>` オプションを用いた名前空間を指定した操作
* Namespace内のPodが `<service-name>.<namespace>.svc.cluster.local` という形式のDNS名で相互に名前解決できることの確認
* `kubectl config set-context` を使ったデフォルトNamespaceの切り替え

### 2-5. ネットワークとサービスディスカバリ（Service / Ingress）
一時的なIPを持つPodへの安定したアクセス経路を構築します。Serviceは **Podのラベル（selector）** に基づいて通信先を決定するため、Pod自体のIPアドレスを意識する必要がありません。kube-proxyが各NodeでServiceのIPルーティング（iptables/IPVS）を実現しています。

* **ClusterIP Service**: クラスタ内部からのトラフィックルーティングの確認
* **NodePort Service**: クラスタ外部（ブラウザ等）からノードのIP・ポートを経由したアクセスの確認（※NodePortは学習・開発用途向けで、本番の入口にはIngressを使用する）
* CoreDNSによるService名の名前解決の確認
* **Ingress**: L7（HTTP/HTTPS）のルーティングを行うリソースです。HostベースまたはPathベースのルーティングにより、単一のエンドポイントから複数のServiceにトラフィックを振り分けます。
  * Ingress Controller（例: `ingress-nginx`）のインストール
  * `host` ベースまたは `path` ベースのルーティングルールを持つIngressリソースの作成
  * ServiceとIngressの役割分担の理解：ServiceはL4(TCP/UDP)、IngressはL7(HTTP)

---

## フェーズ3：設定・ストレージ・セキュリティ・スケジューリング

より実践的なアプリケーションの実行に必要な要素を学びます。

### 3-1. アプリケーション設定の分離
* **ConfigMap**: 環境非依存の設定ファイル作成と、Podの環境変数・ボリュームとしてのマウント
* **Secret**: パスワードやAPIキーなど機密データの安全な保存とマウント  
  :::warning SecretはBase64エンコードであり暗号化ではない  
  `kubectl get secret -o yaml` で確認すると、値はBase64でエンコードされているだけです。etcdへの保存時に暗号化するには別途`EncryptionConfiguration`の設定が必要です。本番環境ではHashiCorp VaultやExternal Secrets Operatorとの連携も検討します。
  :::

### 3-2. データ永続化（Storage）
コンテナが破棄されてもデータが消えない仕組みを構築します。

* **PV/PVCと静的プロビジョニング**: PersistentVolume (PV)（管理者がストレージを事前に定義）とPersistentVolumeClaim (PVC)（ユーザーがストレージを要求）の作成と紐付け
* PodへのPVCのマウントと、Podを意図的に削除・再作成した際のデータ永続性の確認
* **StorageClassと動的プロビジョニング**: StorageClassを使用すると、PVCの作成時にPVが自動的に作成されます（Dynamic Provisioning）。`kubectl get storageclass` でクラスタが持つStorageClassを確認します。

### 3-3. セキュリティとアクセス制御（RBAC）
RBACは「**誰が（Subject）**・**何のリソースに（Resource）**・**何をできるか（Verb）**」の3要素で権限を定義します。

* ServiceAccountの作成（PodがKubernetes APIにアクセスする際のIDとなる）
* **Role / ClusterRole**: RoleはNamespaceスコープの権限定義、ClusterRoleはクラスタ全体スコープの権限定義です
* **RoleBinding / ClusterRoleBinding**: SubjectとRole(ClusterRole)を紐付けて権限を付与します
* Roleを特定のServiceAccountに紐付け、そのServiceAccountを使うPodがNamespace内のPodリストのみ取得できる（それ以外はできない）最小権限を体験

### 3-4. リソース管理（ResourceQuota / LimitRange）
Namespaceに対してリソース使用量の上限を設定し、複数チームやアプリケーションが共存するクラスタでの公平なリソース配分を実現します。

* **ResourceQuota**: Namespace全体で使用できるCPU・メモリ・Pod数などの総量を制限します
  ```yaml
  # 例: Namespace全体で最大CPU=4、メモリ=8Gi、Pod数=10
  spec:
    hard:
      requests.cpu: "4"
      limits.memory: "8Gi"
      count/pods: "10"
  ```
* **LimitRange**: Namespace内の個々のPod/Containerに対するデフォルト値と上下限を設定します。ResourceQuotaと組み合わせることで、`requests`/`limits`の記述がないPodにもデフォルト値を適用できます

### 3-5. ネットワークセキュリティ（NetworkPolicy）
デフォルトのKubernetesはNamespaceをまたいだPod間の全通信を許可します（default-allow）。NetworkPolicyを使うことで、L3/L4レベルでPod間の通信を制限できます。ただしNetworkPolicyの施行はCNIプラグインが行うため、Calicoなど対応したCNIが必要です。

* **default-deny**: 全インバウンド通信を拒否するNetworkPolicyを適用し、通信が遮断されることの確認
* **ホワイトリスト追加**: `podSelector` でラベルが一致するPodからの通信のみを許可するNetworkPolicyを追加し、意図したPodからのみアクセスできることの確認
* Namespaceをまたいだ通信制御（`namespaceSelector`）の動作確認

### 3-6. Podスケジューリングの制御
kube-schedulerはNodeのリソース空き状況などを考慮してPodを配置しますが、以下の仕組みを使って配置を明示的に制御できます。

* **nodeSelector**: Podマニフェストに `nodeSelector` フィールドを追加し、特定のラベルを持つNodeにのみスケジュールされることを確認（例: `disktype=ssd` のNodeにのみ配置）
* **Taint / Toleration**: NodeにTaint（汚染）を付与すると、そのTaintを許容する`Toleration`を持つPodしかスケジュールされなくなります。GPU専用ノードや管理用ノードへの一般Podの混入を防ぐ際に使用します
  * `kubectl taint nodes <node> key=value:NoSchedule` でTaintを付与
  * Podマニフェストの `tolerations` フィールドでTaintを許容
* **NodeAffinity**: `nodeSelector` より表現力が高く、「このラベルのNodeを優先する（ソフト）」「このラベルのNodeにのみ配置する（ハード）」を `requiredDuringSchedulingIgnoredDuringExecution` / `preferredDuringSchedulingIgnoredDuringExecution` で使い分けられます

---

## フェーズ4：クラウドネイティブ・エコシステムの体験

「クラウドネイティブアーキテクチャ（16%）」「可観測性（8%）」「アプリケーションの配信（8%）」の範囲に対応します。CNCF Landscapeにある主要ツールの役割を体感します。

### 4-1. コンテナイメージのビルドとレジストリ
アプリケーション配信の出発点は、コンテナイメージの作成とレジストリへの保存です。

* **Dockerfile / Containerfile** を用いたカスタムイメージのビルド (`docker build` または `podman build`)
* タグ付け（`name:tag` 形式）と、タグとは別にイメージの内容を一意に識別する **ダイジェスト（`@sha256:...`）** の確認
* Docker HubやGitHub Container Registry (ghcr.io) などのコンテナレジストリへのpush
* KubernetesのPodマニフェストからプライベートレジストリのイメージをpullするための `imagePullSecret` の設定

### 4-2. パッケージ管理（Helm）
Kubernetesマニフェストをテンプレート化し、バージョン管理されたパッケージ（Chart）として配布・管理する仕組みです。

* Helm CLIのインストール
* 公開チャート（例: bitnami/nginx, bitnami/redis）の検索とインストール
* **`values.yaml`** によるカスタマイズ（デフォルト値の上書き）と `--set` オプションの使い方
* Helmによるリリース管理（アップグレード、アンインストール）

### 4-3. 可観測性（Observability）
システムの内部状態を把握するための「テレメトリの3本柱」を理解します。

:::info 可観測性の3本柱
| 種類 | 問いかけ | 代表ツール |
|---|---|---|
| **Metrics（メトリクス）** | 「どのくらい？」 — CPUやメモリなど数値データの時系列変化 | Prometheus, Grafana |
| **Logs（ログ）** | 「何が起きた？」 — イベントやエラーの記録 | Loki, Fluent Bit, Elasticsearch |
| **Traces（トレース）** | 「どこで遅い？」 — リクエストのサービス間伝播の追跡 | Jaeger, Zipkin, OpenTelemetry |
:::

* **kubectl によるログ確認**: `kubectl logs`、`kubectl describe`、`kubectl get events` を用いたトラブルシューティングの基礎
* **Metrics**: Helmを利用した **Prometheus**（メトリクス収集）と **Grafana**（可視化）のデプロイ。KubernetesノードやPodのCPU/メモリ使用量ダッシュボードの閲覧
* **Logs**: **Loki**（ログ収集・保存）と **Fluent Bit**（ログ転送エージェント・DaemonSetとして全Nodeに配置）のデプロイ。GrafanaからLokiをデータソースとして追加し、`kubectl logs` と同じログをGrafanaで検索・閲覧する

### 4-4. CI/CDパイプラインの基礎
アプリケーションの変更をクラスタへ安全かつ継続的に反映する仕組みです。

:::info CIとCDの違い
* **CI（継続的インテグレーション）**: コードのpushをトリガーに、自動でビルド・テスト・イメージ作成までを行います
* **CD（継続的デリバリー/デプロイ）**: CIで作成されたアーティファクト（イメージ等）をクラスタへ自動で反映します
:::

* **GitHub Actions** を使ったCI/CDパイプラインの作成
  1. コードのpushをトリガーに自動テストを実行
  2. テスト通過後、Dockerイメージをビルドしてレジストリにpush
  3. Kubernetesマニフェストの `image` タグを新しいダイジェストに更新し、Git Commitする（次のGitOpsフェーズとの連携）

### 4-5. アプリケーションの配信（GitOps）
GitOpsは「GitをKubernetesクラスタの唯一の情報源（Single Source of Truth）とし、Git上のDesired StateとクラスタのActual Stateを常に同期させる」CDアプローチです。CI/CDのCD部分のモダンな実装方式です。

| 従来のpush型CD | GitOps（pull型CD） |
|---|---|
| CIパイプラインが `kubectl apply` を実行（クラスタに直接push） | ArgoCDなどがGitを常に監視し、差分を自動でクラスタに同期（pull） |
| クラスタへの認証情報がCI側に必要 | CI側にクラスタ認証情報が不要（セキュリティ向上） |
| 変更の取り消しは手動 | Gitのrevertで確実にロールバック可能 |

* **ArgoCD** のクラスタへのインストールと初期パスワードの取得
* GitHub等のGitリポジトリにあるマニフェストとArgoCDを連携させ、Gitの変更が自動的にクラスタに同期される（GitOps）状態の確認
* Gitリポジトリのマニフェストを意図的にクラスタの状態と乖離させ、ArgoCDが `OutOfSync` を検知して自動修復することの確認（Self-healing）

### 4-6. サービスメッシュ（概要レベル）
マイクロサービス間の通信に透過的にサイドカープロキシを挿入し、通信制御・観測・セキュリティ（mTLS）をアプリケーションコードから分離して実現する仕組みです。KCNA-JPではサービスメッシュの概念理解が問われるため、深い操作習得よりも全体像の把握を優先します。

* **Linkerd** または **Istio** の最小構成でのインストール
* 既存のアプリケーションにサイドカープロキシを注入し、ダッシュボードからトラフィックの成功率や遅延が可視化されることの確認
