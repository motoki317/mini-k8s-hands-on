# 初めてのコントローラー

## コントローラーとは

[./0_kubernetes_workings.md](./0_kubernetes_workings.md) でも説明しましたが、簡単に言えば Kubernetes は **KVS**（**Key Value Store**）と**コントローラー**から成ります。

KVS の部分では、いくつかリソースの「種類」があり（例: Pod, Deployment, Service, ...）、名前に紐づくリソースの定義（yaml で書くことが多い）を管理しているだけです。

コントローラーにはいくつか種類があり（例: Deployment Controller）、それぞれのコントローラーはある「上位の」リソース（例: Deployment）を監視し、そこから導かれる「下位の」リソース（例: ReplicaSet）を KVS に登録または削除（管理）します。
いくつかのコントローラー達が協調することにより、クラスターの状態は「理想状態」に近づき、最終的には Pod が定義されコンテナ達がデプロイされます。[^1]

[^1]: https://kubernetes.io/docs/concepts/workloads/controllers/

![argocd objects](../images/0_argocd_objects.png)

↑ArgoCDの管理画面。「上位」と「下位」のリソースと、その対応関係が視覚的に理解できる。"deploy"はDeployment, "sts"はStatefulSet, "rs"はReplicaSetの略。

### なぜコントローラーを使うか？

コントローラーと、それに対応する「上位」のリソースを使うことで、複数の「下位の」リソースをまとめて管理できます。

例えば、大量のトラフィックを捌くため、たくさんの Pod をデプロイしたい時があります。
前ページのような yaml をたくさんコピペして（場合によっては100個以上！）、`caddy-1`, `caddy-2` とそれぞれ Pod に名付け、管理するのは大変です。
（複数 Pod に同じ名前を付けることはできません。KVS で管理する時にどの Pod を指すか分からなくなるため。）

ここで Deployment リソースを使うことで、人間が管理するリソースは一つの Deployment だけとなります。
Deployment は複数の ReplicaSet リソースを管理し、ReplicaSet は複数の Pod リソースを管理します。

また、イメージのバージョンを更新する際に、Deployment 等のリソースで細かくアップデート方法（一度に入れ替える Pod の割合等）を制御できます。
これを上手く用いることで、ダウンタイムゼロのWebサイトの更新などができます。

## Deployment とは

Deployment リソースは、**ステートレス**な Pod を管理するのに最もよく使われるリソースです。
簡単に Pod を複製できたり、バージョン更新やその方法を細かく管理できます。

**ステートレス**とは、他の同一 Pod（コンテナ）と「入れ替えが効く」ような Pod（コンテナ）のことです。
言い換えれば、何個コンテナが複製されても問題無いようなアプリケーションのことです。
例えば、大量のHTTPリクエストを処理する、シンプルなAPIサーバーやWebサーバーなどです。

反対に、**ステートフル**とは、アプリケーションに状態（ステート）があり、「入れ替えが効かない」Pod（コンテナ）のことです。
言い換えると、同じ役割のコンテナが複数個存在すると、上手く動かないアプリケーションのことです。
例えば、MySQL などのデータベースサーバーは同じファイルを操作するので、同じ役割のコンテナ（プロセス）が複数個いると上手く動きません。

## Deployment を使う

前ページで使った [caddy](https://hub.docker.com/_/caddy) は、ステートレスな Web サーバーとして使えます。
大量のHTTPリクエストを捌くために複数のWebサーバー（Pod）を建てたいと仮定し、これを実現する最小構成の Deployment を書きます。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: caddy

spec:
  # 3つのレプリカ（Pod）を定義
  replicas: 3

  # 管理対象とする Pod のセレクター
  selector:
    matchLabels:
      app: caddy

  # 各 Pod のテンプレート（通常の Pod 定義とだいたい同じ）
  template:
    # ここで上のセレクターと同じラベルを付ける
    metadata:
      labels:
        app: caddy
    spec:
      containers:
        - name: caddy
          image: caddy:2.6
```

[../examples/4_deployment.yaml](../examples/4_deployment.yaml)

Pod 単体よりは少し定義が複雑になっていますね。
それぞれのフィールドを解説します。

- `.spec.replicas` : Pod のレプリカ数を定義します。3 と書けば同一の Pod が 3 個デプロイされるし、100 と書けば 100 個デプロイされます。
- `.spec.selector` : 管理対象とする Pod の「セレクター」を定義します。Deployment Controller はこのセレクターを用いてクラスター内の Pod を列挙し、それらを管理対象とします。その数が `.spec.replicas` 以上であればいくつかを削除、以下であれば新しく Pod を作成します。
- `.spec.template` : Deployment Controller が Pod を新しく作成する際に、テンプレートとする Pod 定義です。前ページの Pod 定義と大部分は同じです。
- `.spec.template.metadata` : `.spec.selector` で選択可能な Pod の「ラベル」を定義します。ここでは `app: caddy` という同一のラベルを定義しています。Deployment Controller は `.spec.selector` で Pod を列挙・管理しているため、定義が一致しないとずっと「Pod が足りない」と判断され、際限なく Pod が作成され続けてしまいます。

正確には Deployment Controller が直接 Pod を管理するのではなく、間に ReplicaSet リソースとその ReplicaSet Controller が挟まりますが[^2]、ここでは説明を簡単にするため、詳細は省略します。
「Deployment リソースを定義すると、その数だけ Pod が作られる」ことが理解できれば大丈夫です。

[^2]: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

さて、Deployment を「デプロイ」していきましょう！
（Do you see the joke?）

次のコマンドで、今書いた Deployment をクラスターに登録します。

- `kubectl apply -f ./examples/4_deployment.yaml`

Pod の時と同じように、クラスターに存在する Deployment 一覧を取得します。

- `kubectl get deployments`

```yaml
$ kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
caddy   3/3     3            3           6s
```

Apply 直後は `READY` の数が少ないかもしれませんが、これも少し待つと `3/3` となります。
宣言的な管理の嬉しさが少し分かってきましたね。

`READY` は、「`Status: Ready` である管理対象 Pod の数（= 立ち上がった Pod の数）」 / 「`.spec/replicas` の数」を示しています。

本当に Pod が 3 つ立ち上がったのでしょうか？確かめます。

- `kubectl get pods`

```yaml
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
caddy-766687f85d-k5wx5   1/1     Running   0          6m7s
caddy-766687f85d-sj2nm   1/1     Running   0          6m7s
caddy-766687f85d-mmkl6   1/1     Running   0          6m7s
```

確かに Pod が 3 つ立ち上がっています。
同一名の Pod は作れないため、Deployment Controller は末尾にランダムな suffix を付けて Pod を作成します。
この管理の仕方からも、各 Pod の**ステートレス**性（入れ替えが効く）を前提にしていることが伺えます。

### アプリケーションにアクセスする

再度、Webサーバーが正常に動いているかを確認します。

次のコマンドで、Deployment の Pod **群**へ port-forward を行います。

- `kubectl port-forward deployment/caddy 8080:80`

「Pod **群**へ port-forward する」と、各HTTPリクエストが 3 つの Pod のうちいずれか 1 つに渡るという挙動になります。
最も基本的なロードバランシングです。
（正確な挙動は、次ページの Service, Ingress の項で説明します。）

https://localhost:8080/ へアクセスし、再度次の画面が見えたら成功です。

![caddy-hello-world](../images/3_caddy_hello_world.png)

### アプリケーションを更新する

次に、アプリケーションのバージョンを更新します。

Deployment リソースを書き換え、再度 apply することで、更新が完了します。

`./examples/4_deployment.yaml` ファイル中の、`.spec.template.spec.containers[0].image` フィールドを書き換えて、イメージのバージョンを更新してみましょう。

元は `caddy:2.6` と指定してありましたが、試しに `caddy:2.7` へ書き換えてみましょう。
（もしくはv2の最新を指す `caddy:2` や `caddy:latest` でも構わない）

次のような YAML になれば正解です。

```yaml
# 前略
      containers:
        - name: caddy
          image: caddy:2.7 # ここを更新して、イメージを更新！
```

バージョン更新の様子が分かるように、別ターミナルで次のコマンドを打って、Pod の状態を監視します。

- `watch -n 0.5 kubectl get pods`
    - 0.5秒ごとに `kubectl get pods` を実行します。

次に、更新したリソース定義を、クラスターに登録します。

- `kubectl apply -f ./examples/4_deployment.yaml`

次のように `configured` と出たら更新された証拠です。
`unchanged` と出たら、YAML をしっかり書き換えて保存したかを再度確認しましょう。

```plaintext
$ kubectl apply -f ./examples/4_deployment.yaml 
deployment.apps/caddy configured
```

更新したら、`kubectl get pods` の様子をよく見てみましょう。

```plaintext
NAME                     READY   STATUS              RESTARTS   AGE
caddy-6977985c98-95x9q   1/1     Running             0          59s
caddy-7b56f884f5-88cjq   1/1     Running             0          2s
caddy-7b56f884f5-7rw7g   1/1     Running             0          1s
caddy-6977985c98-wwjrw   1/1     Terminating         0          59s
caddy-7b56f884f5-29gtw   0/1     ContainerCreating   0          0s
```

分かりづらいかもしれませんが、「一気に古い3つ Pod が終了し、一気に新しい3つの Pod が作られる」ではなく、「少しずつ古い Pod が終了し、少しずつ新しい Pod が作られる」ことに気づきましたか？
見落としてしまったら、`caddy:2.6` と `caddy:2.7` を行き来させて、何度も確認してみましょう。

Deployment Controller はデフォルトで、アップデート中も `.spec.replicas` の 75% 以上の Pod が常に `Running` であることを保証（しようと）します。
75% という細かい数値は Deployment リソースの定義で調整可能ですが、このアップデート機構を上手く用いることで、ゼロダウンタイムでのWebサイト更新ができます。

### お掃除

確認を終えたら、Deployment を削除しておきましょう。

- `kubectl delete deployment caddy`

Deployment と Pod が消えることを確認します。

- `kubectl get deployments`
- `kubectl get pods`

## StatefulSet を使う

Deployment ではステートレスな Pod を管理しましたが、
**ステートフル**な（入れ替えが効かない、同じ役割のコンテナが存在するとまずい）Pod は **StatefulSet** を用いて管理するのが一般的です。

StatefulSet リソースでは、1個以上のステートフルな Pod と、その更新順序などを細かく管理できます。

さて、ここでは [redis](https://hub.docker.com/_/redis) をステートフルなアプリケーションの例として使います。
（Redis は内部で状態を持っており、特に永続化する場合は、同じ役割のコンテナが同時に存在すると困ります。）

1 つの Redis の Pod を管理する、最小の StatefulSet リソースを書いていきます。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis

spec:
  # StatefulSet が管理する Pod にアクセスするための Service 名
  serviceName: redis

  # 1つのレプリカ（Pod）を定義
  replicas: 1

  # 管理対象とする Pod のセレクター
  selector:
    matchLabels:
      app: redis

  # 各 Pod のテンプレート
  template:
    # ここで上のセレクターと同じラベルを付ける
    metadata:
      labels:
        app: redis
    # 以下は通常の Pod の定義
    spec:
      containers:
        - name: redis
          image: redis:7

---
# StatefulSet にアクセスするための Service
apiVersion: v1
kind: Service
metadata:
  name: redis

spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
```

Deployment と同じフィールドは説明を省きますが、一つ必須なフィールドが増えました。
また、`---` で区切られて、Service というリソースが同時に定義されています。

- `.spec.serviceName` : Pod にアクセスするための Service 名。クラスター内コンテナから、StatefulSet に通信する時は、このサービス名を通して解決する。（例: `redis:6379`）
- Service : クラスター内コンテナから、Pod に直IP以外でアクセスするために必要。Service について詳しくは後述。

天下り的に Service が少し出てきてましたが、StatefulSet の本質ではないため、詳しくは後で解説します。

次のコマンドで、StatefulSet と Service をクラスターに登録します。

- `kubectl apply -f ./examples/4_statefulset.yaml`
    - `---` で区切っているため、複数リソースを1ファイルから登録できます。

StatefulSet と Service が登録されたことを確認します。

- `kubectl get statefulsets`
- `kubectl get services`
    - 2個出てきますが、`kubernetes` Service はデフォルトで入っているもので、正常です。

```plaintext
$ kubectl get statefulsets
NAME    READY   AGE
redis   1/1     5m4s

$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP    4h46m
redis        ClusterIP   10.43.122.204   <none>        6379/TCP   5m6s
```

また、StatefulSet によって Pod が一つ作成されたことを確認します。

- `kubectl get pods`

```plaintext
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
redis-0   1/1     Running   0          5m52s
```

Pod の名前は、Deployment のランダムな suffix の管理と違い、`{StatefulSetの名前}-{番号}` といった予測可能な名前になっています。
この管理の仕方からも、同じ役割の Pod が同時に2つ存在できないこと（同じ名前の Pod は存在できないことを思い出してください）が伺えます。
実際に、StatefulSet が Pod を更新する時は、今ある Pod（例: `redis-0`）を削除してから、新しい Pod（例: `redis-0`）を作成します。

### アプリケーションを更新する

実際に、同じ役割の Pod が同時に存在しない（この例では、redis の Pod が同時に2個以上存在しない）ことを確認しましょう。

更新の様子が分かるように、別ターミナルで次のコマンドを打って、Pod の状態を監視します。

- `watch -n 0.5 kubectl get pods`

別のターミナルを開き、次のコマンドで StatefulSet の更新を模した、再起動を行います。

- `kubectl rollout restart statefulset/redis`

一瞬ですが、Pod の状態がまず `Terminating` になってから、次の Pod が`ContainerCreating` になり、最後に `Running` になったことが分かったでしょうか。
同時に `redis-0` のような Pod が2個存在する瞬間が存在しないことを確認しましょう。

### アプリケーションにアクセスする

最後に、アプリケーションが正常に動いているかを確認します。

次のコマンドで、StatefulSet が管理する Pod **群**へ port-forward を行います。

- `kubectl port-forward statefulset/redis 6379:6379`

別ターミナルから次のコマンドで、redis-cli で redis へアクセスします。

- `docker run -it --rm --network host redis redis-cli -h localhost`
    - もしくは redis-cli が手元にある場合は、単に `redis-cli -h localhost`

次のような動作が確認できれば成功です。

```plaintext
$ docker run -it --rm --network host redis redis-cli -h localhost
localhost:6379> SET foo bar
OK
localhost:6379> GET foo
"bar"
localhost:6379> KEYS *
1) "foo"
```

### お掃除

StatefulSet と Service をクラスターから削除します。

- 方法1
    - `kubectl delete statefulset redis`
    - `kubectl delete service redis`
- 方法2
    - `kubectl delete -f ./examples/4_statefulset.yaml`

## 次へ

このページでは、（複数の）コンテナをデプロイ・バージョン管理する、一般的な方法を学びました。

次では、実際にこれらの Pod 群にアクセスするための方法を学びます。

[./5_service.md](./5_services.md)
