# 最初のコンテナ

## Pod とは

Kubernetes でコンテナをデプロイする一番原始的な方法は、Pod というリソースを定義することです。

Pod は、複数のコンテナ（docker コンテナと同等と思ってもらって良いです）をまとめたものです。
プロセス（PID）空間やファイルシステム空間は、docker と同様に、コンテナごとに分離されます。

しかし Kubernetes におけるネットワークの分離単位は Pod であり、
これは裏返すと一つの Pod の中ではネットワークが分離されない、ということです。
したがって、同 Pod のコンテナはお互いに `localhost:ポート番号` で通信できます。

ただし、最初のうちは複数コンテナを詰め込むことはあまり無いので、「1 Pod ≒ 1 docker コンテナ」といった理解で問題無いです。

## 最初の Pod

Pod の最小の定義は以下になります。

> [!NOTE]
> 正常にWebサーバーがデプロイできたことを確認するために、[caddy](https://hub.docker.com/_/caddy) のイメージを使いますが、お好みの docker image でも良いでしょう。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: caddy

spec:
  containers:
    - name: caddy
      image: caddy:2
```

[../examples/3_pod.yaml](../examples/3_pod.yaml)

これをファイルに書き出して、クラスターに Pod を登録しましょう！

- `kubectl apply -f ./examples/3_pod.yaml`
    - このリポジトリの `./examples` フォルダーに必要なファイルをまとめてあります。自分でファイルを作成した場合は、そのパスを指定しましょう。

次のコマンドで、クラスターに存在する Pod 一覧を取得します。

- `kubectl get pods`

次のような出力が出ると成功です。

```plaintext
$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
caddy   1/1     Running   0          12m
```

最初は `STATUS` が `Pending` などと出るかもしれませんが、数秒から数十秒で自動的に `Running` になるはずです。
こうした「理想状態への自動的な漸近・回復」は、Kubernetes の最も便利な特徴の一つです。

開発者は「Pod が一つ動いている」という理想状態を yaml で表記しクラスターに登録するだけで、残りは Kubernetes がこの理想状態を実現してくれます。
Docker でいう、`docker pull` + `docker create` + `docker run` の操作を Kubernetes が代わりに実行してくれます。

## Pod を削除する

次に、今作った Pod を削除してみましょう。

「いきなり？」と思うかもしれませんが、Kubernetes の「理想状態を宣言するだけ」という理念を理解するためです。
この理念によって、どれだけコンテナの管理が楽になるかを見てみましょう。

次のコマンドで Pod を API Server から削除します。

- `kubectl delete pods caddy`

Pod 一覧を再度確認して、次のような出力が出ると成功です。

```plaintext
$ kubectl get pods
No resources found in default namespace.
```

上のコマンドで、開発者（あなた）は「Pod が1つ動いている」という状態を、クラスターから削除しました。
クラスターの理想状態は「Pod は何も動いていない」にという状態になり、Kubernetes はこの状態を実現するため、実際にコンテナを削除します。
Docker でいう、`docker stop` + `docker rm` の操作を Kubernetes が代わりに実行してくれます。

Docker では、コンテナ一つ一つを「手続き的に」管理するために、`run` や `stop` といったコマンドを発行していましたが、
Kubernetes では、大量のコンテナを一括で「**宣言的に**」管理するために、「**理想状態の登録と削除**」**しか行っていない**ことに気づいたでしょうか。

Pod 1つだけではあまり実感が沸かないかもしれませんが、コンテナの種類や数が増えると、この宣言的な管理の重要性が分かるかもしれません。
本当に**人間が欲しい価値**は「**コンテナが動いている**」状態そのものであり、それを実現するための手続き、またホストの再起動や障害などで理想状態から離れた（"ドリフト"した）ときの自動的な回復を、**機械が自動的にやってくれる**のです。

さて、簡単に管理ができることが分かったところで、再度 Pod をクラスターに登録しておきましょう。

- `kubectl apply -f ./examples/3_pod.yaml`

## アプリケーションにアクセスする

コンテナが正常に動いているかを確認しましょう。

次のコマンドで、Pod 内のポートに接続できます。

- `kubectl port-forward pod/caddy 8080:80`
    - `caddy` という名前の Pod の 80 番ポートに、ローカルの 8080 番ポートから接続できるようにする、という意味です。

次のような出力が出たら、port-forward が成功しています。
コマンドは終了しませんが、これで正常です。

```plaintext
$ kubectl port-forward pod/caddy 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

https://localhost:8080/ にブラウザからアクセスし、次のWebページが見えたら成功です。

![caddy-hello-world](../images/caddy-hello-world.png)

ページが確認できたら、port-forward コマンドを CTRL+C で終了します。

次の段階へゴミを残さないために、Pod をクラスターから削除しておきましょう。

- `kubectl delete pod caddy`

## 次へ

Pod と基本的な Kubernetes の理念が理解できたら、
より一般的なデプロイ方法を学びましょう！

[./4_controllers.md](./4_controllers.md)
