---
title: Kubernetes の基本リソース操作
description: Pod、Deployment、Service、Namespace などの Kubernetes の基本リソース操作と宣言的管理の操作を行います
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Kubernetesの基本リソース操作
===

> ## Pod の基本操作
> 
> ### Pod の実行 (命令的 vs 宣言的)
> 
> ```bash
> # 命令的 (Imperative): コマンドで直接実行
> kubectl run nginx-pod --image=nginx:alpine
> ---
> # 宣言的 (Declarative): YAMLを定義して適用
> cat <<EOF > nginx-pod.yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: nginx-pod-manifest
>   labels:
>     app: nginx
> spec:
>   containers:
>   - name: nginx
>     image: nginx:alpine
>     ports:
>     - containerPort: 80   # コンテナが Listen するポート
>       hostPort: 8080      # ホストの 8080 番をコンテナの 80 番に転送（ブラウザから http://localhost:8080 でアクセス可能）
>     resources:
>       requests:
>         memory: "64Mi"   # スケジューラがNode選択に使う最低保証量
>         cpu: "250m"
>       limits:
>         memory: "128Mi"  # 超過するとOOMKillされる上限量
>         cpu: "500m"
>     livenessProbe:        # 失敗するとコンテナを再起動
>       httpGet:
>         path: /
>         port: 80
>       initialDelaySeconds: 5
>       periodSeconds: 10
>     readinessProbe:       # 失敗するとServiceのエンドポイントから除外
>       httpGet:
>         path: /
>         port: 80
>       initialDelaySeconds: 3
>       periodSeconds: 5
> EOF
> 
> kubectl apply -f nginx-pod.yaml
> ```
> 
> ### Pod の状態確認とトラブルシューティング
> 
> ```bash
> # Pod一覧の表示
> kubectl get pods
> 
> # Pod状態の変化をリアルタイムで監視 (Pending→ContainerCreating→Running を観察)
> kubectl get pods --watch
> 
> # 詳細情報の確認（Events欄で障害原因を特定する際に必須）
> kubectl describe pod nginx-pod-manifest
> 
> # クラスタ全体のイベントを時系列で確認
> kubectl get events --sort-by='.lastTimestamp'
> 
> # ログの表示
> kubectl logs nginx-pod-manifest
> 
> # Pod内でのコマンド実行
> kubectl exec -it nginx-pod-manifest -- sh
> 
> # ⚠️ 単体Podには「停止」コマンドは存在しない。削除のみ可能。
> # 「停止して再開したい」場合はDeploymentを使い、以下でレプリカ数を0にする（別セクション参照）
> # kubectl scale deployment <deployment名> --replicas=0
> 
> # Podを削除する（マニフェストから起動した場合）
> kubectl delete -f nginx-pod.yaml
> 
> # Pod名を指定して削除する
> kubectl delete pod nginx-pod-manifest
> ```
> 
> ---
> 
> ## Deployment によるワークロード管理
> 
> Deployment は ReplicaSet を介して Pod のレプリカ数やアップデートを管理します。
> 
> ```mermaid
> graph TD
>     Deploy[Deployment] -->|manages| RS[ReplicaSet]
>     RS -->|creates/deletes| Pod1[Pod]
>     RS --> Pod2[Pod]
>     RS --> Pod3[Pod]
> ```
> 
> ### Deployment の作成とスケーリング
> 
> ```bash
> cat <<EOF > deployment.yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: web-deploy
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: web
>   template:
>     metadata:
>       labels:
>         app: web
>     spec:
>       containers:
>       - name: nginx
>         image: nginx:1.25
>         ports:
>         - containerPort: 80
> EOF
> 
> kubectl apply -f deployment.yaml
> 
> # Deployment → ReplicaSet → Pod の3層構造を確認
> kubectl get deployment,replicaset,pod -l app=web
> 
> # レプリカ数の変更 (スケールアウト)
> kubectl scale deployment web-deploy --replicas=5
> ```
> 
> この段階では Pod が3個、それぞれの Pod にコンテナが 1 個ずつ起動しますが、curl コマンドやブラウザで参照しても nginx に接続できません。
> 
> 
> #### Service オブジェクトを作成してロードバランスする
> 
> ```yaml title="service.yaml"
> apiVersion: v1
> kind: Service
> metadata:
>   name: web-service
> spec:
>   type: NodePort # ★ここを環境に合わせて変える（下記参照）
>   selector:
>     app: web     # ★重要：これがあることで「app=web」というラベルがついた3つのPodを探し出して紐付けます
>   ports:
>     - protocol: TCP
>       port: 80        # Service自体が受けるポート
>       targetPort: 80  # Pod（コンテナ）のポート
> ```
> 
> ```bash
> # Service オブジェクトの作成
> kubectl apply -f service.yaml
> 
> # Service の確認
> kubectl get service web-service
> > NAME          TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
> > web-service   NodePort   10.111.225.120   <none>        80:31313/TCP   11s
> ```
> 
> Worker ノード上で curl コマンドを `localhost:31313` に接続すると nginx につながるものの、リモート PC から参照できない。
> 
> #### Ingress コントローラーの導入
> 
> - Ingress コントローラーのインストール (Bare Metal 環境向け)
> 
>     ```bash
>     # NGINX Ingress Controller のインストール
>     kubectl apply \
>         -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml
> 
>     # NodePort を 8080 番に固定する
>     # 💡 成功しない場合: デフォルトの NodePort 範囲 (30000-32767) 外である可能性があります。
>     # その場合は 30080 等を使用するか、API Server の設定変更が必要です。
>     kubectl patch service ingress-nginx-controller -n ingress-nginx \
>       --type='json' \
>       -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30080}]'
> 
>     # 起動確認 (Running になるまで待ちます)
>     kubectl get pods -n ingress-nginx
>     ```
> 
> - Ingress の設定
> 
>     ```yaml title="ingress.yaml"
>     apiVersion: networking.k8s.io/v1
>     kind: Ingress
>     metadata:
>       name: web-ingress
>       annotations:
>         # プレフィックスの書き換え設定（必要に応じて）
>         nginx.ingress.kubernetes.io/rewrite-target: /
>     spec:
>       ingressClassName: nginx # 使用する Ingress Controller の指定
>       rules:
>       - http:
>           paths:
>           - path: /
>             pathType: Prefix
>             backend:
>               service:
>                 name: web-service # 転送先の Service 名
>                 port:
>                   number: 80      # Service が待ち受けているポート
>     ```
> 
> - Ingress の設定反映
> 
>     ```bash
>     kubectl apply -f ingress.yaml
>     ```
> 
> #### Ingress の接続確認と管理
> 
> Ingress Controller は、外部からのリクエストを NodePort 経由で受け取り、各 Service へ振り分けます。
> 
> - **接続先ポートの確認**
> 
>     Bare Metal 環境では、`ingress-nginx-controller` という Service が作成され、そこに割り当てられた NodePort を使用してアクセスします。
> 
>     ```bash
>     kubectl get service -n ingress-nginx ingress-nginx-controller
>     # 実行結果例:
>     # NAME                       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
>     # ingress-nginx-controller   NodePort   10.108.133.31   <none>        80:30080/TCP,443:30544/TCP    2m
>     ```
>     上記例の場合、`http://<NodeのIPアドレス>:30080` でアクセス可能になります。
> 
> - **設定の確認 (Check)**
> 
>     ```bash
>     # Ingress 一覧の表示
>     kubectl get ingress
> 
>     # 詳細な設定内容とイベントの確認
>     kubectl describe ingress web-ingress
>     ```
> 
> - **設定の更新 (Update)**
> 
>     ```bash
>     # マニフェストファイルを修正して再適用
>     kubectl apply -f ingress.yaml
> 
>     # または直接エディタで編集
>     # kubectl edit ingress web-ingress
>     ```
> 
> - **設定の削除 (Delete)**
> 
>     ```bash
>     # マニフェストファイルを指定して削除
>     kubectl delete -f ingress.yaml
> 
>     # または名前を指定して削除
>     # kubectl delete ingress web-ingress
>     ```
> 
> #### ポート 30080 ⇒ 80 にしたい
> 
> Kubernetes ではデフォルトで「30000〜32767」の範囲のポートが設定され、この範囲外のポートを設定しようとするとエラーになります。  
> この制限は Kubernetes の設定で変更できます。
> 
> - /etc/kubernetes/manifests/kube-apiserver.yaml の変更
> 
>     ```diff title="/etc/kubernetes/manifests/kube-apiserver.yaml"
>          - --service-cluster-ip-range=10.96.0.0/12
>          - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
>          - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
>     +    - --service-node-port-range=80-32767
>          image: registry.k8s.io/kube-apiserver:v1.36.1
>          imagePullPolicy: IfNotPresent
>          livenessProbe:
>     ```
> 
>     設定ファイルを保存すると Kubernetes は自動で再読み込み (もしかしたら再起動) します。
> 
> - Ingress の待受けポートの変更
> 
>     ```bash
>     kubectl patch service ingress-nginx-controller -n ingress-nginx \
>         --type='json' \
>         -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value": 80}]'
>     ```
> 
> - 待受けポート変更確認
> 
>     ```bash
>     kubectl get service -n ingress-nginx ingress-nginx-controller
>     ```
> 
> #### ロードバランスの確認
> 
> ```bash
> kubectl exec -it web-deploy-57794f8f99-44db8 \
>     -- bash -c "echo \"server1\" > /usr/share/nginx/html/index.html"
> kubectl exec -it web-deploy-57794f8f99-vp2t7 \
>     -- bash -c "echo \"server2\" > /usr/share/nginx/html/index.html"
> kubectl exec -it web-deploy-57794f8f99-ws2dd \
>     -- bash -c "echo \"server3\" > /usr/share/nginx/html/index.html"
> ```
> 


### Self-healing の確認

Deployment が管理する Pod は、意図せず削除されても自動的に再作成されます。

```bash
# Podを1つ強制削除
kubectl delete $(kubectl get pod -l app=web -o name | head -1)

# 自動復旧の様子を監視 (READY数が一時的に減り、すぐに回復する)
kubectl get pods -l app=web --watch
```

### ローリングアップデートとロールバック

```bash
# イメージの更新 (v1.25 -> v1.26)
kubectl set image deployment/web-deploy nginx=nginx:1.26

# アップデート状況の確認
kubectl rollout status deployment/web-deploy

# 履歴の確認
kubectl rollout history deployment/web-deploy

# ロールバック (以前のバージョンに戻す)
kubectl rollout undo deployment/web-deploy
```

### deployment 関連コマンド

```bash
# Deployment の削除
kubectl delete deployment/web-deploy
```

---

## 用途別ワークロードリソース

Deploymentの他にも、目的に応じた複数のワークロードリソースがあります。

### StatefulSet
Podに安定したネットワークID（hostname）と専用の永続ストレージを提供します。Pod名が `name-0`, `name-1` と順序付けられ、再作成後も同じ名前・PVCが紐付きます。MySQLやZooKeeperなどのステートフルなアプリケーションに使用します。

```bash
cat <<EOF > statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-sts
spec:
  serviceName: "web-sts-svc"   # Headless Serviceへの参照（DNS名の解決に使用）
  replicas: 3
  selector:
    matchLabels:
      app: web-sts
  template:
    metadata:
      labels:
        app: web-sts
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:         # Podごとに個別のPVCが自動作成される
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
EOF

kubectl apply -f statefulset.yaml

# Pod名が web-sts-0, web-sts-1, web-sts-2 と順序付けられていることを確認
kubectl get pods -l app=web-sts

# 各Podに専用のPVCが自動作成されていることを確認
kubectl get pvc
```

### DaemonSet

クラスタの全 Node (または指定条件を満たすNode) に1つずつPodを配置します。  
新しい Node が追加されると自動で Pod が起動します。  
ログ収集エージェントやノード監視エージェントに使用します。

```bash
cat <<EOF > daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
      - name: log-agent
        image: busybox
        command: ["sh", "-c", "while true; do echo 'Collecting logs...'; sleep 60; done"]
EOF

kubectl apply -f daemonset.yaml

# 全NodeにPodが1つずつ配置されていることを確認 (-o wide でどのNodeかも表示)
kubectl get pods -l app=log-agent -o wide
```

### Job

処理が完了 (exit code 0) することを目的とするワークロードです。  
処理が完了しても Pod は再起動されません。  
バッチ処理や DB マイグレーションに使用します。

```bash
cat <<EOF > job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo 'Batch job completed'; date"]
      restartPolicy: Never      # JobではOnFailureまたはNeverを指定する
EOF

kubectl apply -f job.yaml

# Jobの完了状態を確認 (COMPLETIONS: 1/1 になること)
kubectl get jobs
kubectl logs -l job-name=batch-job
```

### CronJob

Job を cron 形式のスケジュールで定期実行します。  
実行のたびに Job リソースと Pod が生成されます。

```bash
cat <<EOF > cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: periodic-task
spec:
  schedule: "*/5 * * * *"    # 5分ごとに実行（動作確認用）
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: task
            image: busybox
            command: ["sh", "-c", "date; echo 'CronJob executed'"]
          restartPolicy: OnFailure
EOF

kubectl apply -f cronjob.yaml

kubectl get cronjobs
# スケジュール実行後にJobとPodが自動生成されることを確認
kubectl get jobs
```

---

## Namespace による分離

Namespace はクラスタを論理的に分割する仕組みです。  
チームや環境 (dev/staging/prod) ごとにリソースを分離します。

```bash
# Namespaceの作成
kubectl create namespace dev-team

# Namespaceを指定してPodを作成
kubectl run nginx-dev --image=nginx:alpine -n dev-team

# 特定のNamespaceのリソースのみ表示
kubectl get pods -n dev-team

# 全NamespaceのPodを一覧表示
kubectl get pods -A

# デフォルトのNamespaceを切り替える
kubectl config set-context --current --namespace=dev-team

# 確認後はdefaultに戻す
kubectl config set-context --current --namespace=default
```

:::tip DNS名前解決
Namespace内のServiceには `<service名>.<namespace>.svc.cluster.local` という形式のDNS名が付与されます。
例: `web-service.dev-team.svc.cluster.local`
:::

---

## Service と Ingress

Serviceは **Podのラベル (selector)** に基づいて通信先を決定するため、Pod の IP アドレスを意識せずに安定したアクセスポイントを提供できます。  
kube-proxy が各 Node で IP ルーティングを実現しています。

### ClusterIP (クラスタ内部通信用)

```bash
kubectl expose deployment web-deploy --port=80 --target-port=80 --name=web-service --type=ClusterIP

# ServiceのCluster IPと、実際のPod IPリスト（Endpoints）を確認
kubectl get svc web-service
kubectl get endpoints web-service
```

### NodePort (クラスタ外部からの簡易アクセス用)

```bash
kubectl expose deployment web-deploy --port=80 --target-port=80 --name=web-nodeport --type=NodePort

# 割り当てられたポートの確認 (30000-32767の範囲)
kubectl get svc web-nodeport

# アクセス確認
# curl http://<NodeのIPアドレス>:<NodePort番号>
```

:::note
NodePortは学習・開発用途向けです。本番環境の入口にはIngressを使用します。
:::

### Ingress (L7ルーティング)

L7 (HTTP/HTTPS) レベルでのルーティングを行います。  
ホスト名やパスに基づいて複数の Service にトラフィックを振り分けます。  
機能するには事前に Ingress Controller のインストールが必要です。

```bash
# Ingress Controller (ingress-nginx) のインストール
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

# NodePort を 8080 番に固定する場合 (Bare Metal 環境で便利)
kubectl patch service ingress-nginx-controller -n ingress-nginx \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value": 8080}]'

# Ingress Controllerの起動を待機
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

```bash
# Ingressリソースの作成 (pathベースのルーティング例)
cat <<EOF > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

kubectl apply -f ingress.yaml

# 状態確認
kubectl get ingress
kubectl describe ingress web-ingress
```
