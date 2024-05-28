# しくみから分かる Kubernetes

## Kubernetes とは

k8s (Kubernetes) とは、オープンソースの **Container Orchestration System** です。
https://kubernetes.io/

複数のノード（VPSとか物理マシンとか）を管理し、コンテナ化されたアプリを人間の代わりに管理してくれます。
まとまった複数のノードは**クラスタ**と呼ばれます。

Dockerなどをそのまま使うのに比べて、大きく2つのメリットがあります。

- **複数種コンテナ**のデプロイを、安定的に**自動化**できる。
- **大量の**トラフィックを**安定的に**捌くために、複数のコンテナを複数ノードに渡ってデプロイできる。

## 簡単な仕組み

誤解を恐れずにいうと、k8sは単なる **KVS** (**key value store**) と**コントローラ**からなります。
以下のような**リソースオブジェクト**をyamlで宣言し、REST APIを通じてKVS（下画像の Kubernetes Objects）にぶちこむことでインフラの状態を「**宣言的に**」管理します。
このyamlの宣言は **manifest** とも呼ばれます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

「宣言的に」とは、「最終的な状態だけを定義しておき、そこに至るまでの具体的な操作は記述しない」ということです。

リソース更新等のイベント発生時や定時実行をトリガーとし、各コントローラが現在の状態と理想状態の差異を計算します。
この差異から、具体的なインフラの操作（下画像の System Resources）や、サブリソース（下画像の Kubernetes Objects）の管理を行います。

この差異の計算と操作のループのことを、"**reconciliation loop**"と呼びます。
シンプルですが、この reconciliation loop が非常に堅牢なことが実証されており、デプロイを安定的に自動化できる仕組みとなっています。

![kubernetes controllers](../images/0_kubernetes_controllers.png)

画像元: https://iximiuz.com/en/series/writing-kubernetes-controllers-operators/

## 基本リソース

すごく簡潔に説明すると、こんな感じ。

- Pod: コンテナを1個以上詰め込める。同一pod内のコンテナは互いにlocalhostでアクセスできる。
    - だいたいdockerのコンテナと同じと思っておけばいい
    - コンテナを複数個詰め込む場面はそんなにない
- Service: Podに外部からTCP/UDP(L4)通信するために必要。複数のpodを選択できる。
    - type: NodePortやLoadBalancerで外部にポートを公開できる
- Ingress: Podに外部からHTTP(L7)通信するために必要。複数のserviceを選択できる。
    - 最近はGateway APIとかいうのもあるらしい

ネットワーク関連(ServiceやIngress)がコンテナの実態(Pod)と切り離されているのは、1個のServiceやIngressで複数個のPodを指定して**ロードバランス**させるためです。

![argocd traffic](../images/0_argocd_traffic.png)

↑ArgoCDの管理画面①。type: LoadBalancerであるServiceが、あるIPに来たトラフィックをPodに渡している様子を視覚的に理解できる。

## リソースの階層構造

普通は直接Podをyamlで定義せず、DeploymentやStatefulSetなど、「上位の」リソースを人間が扱います。
Deployment Controllerなどの各コントローラが、Deploymentなどの「上位の」リソースを監視し、ReplicaSet, Podなどの「下位の」リソースを管理します。

Podが単にコンテナ1つを管理するのに対して、DeploymentやStatefulSetはそのレプリカ数、ノードが壊れた場合のPodの再起動・再設置、バージョン管理、起動の順番の管理などを行う、といった具合です。

![argocd objects](../images/0_argocd_objects.png)

↑ArgoCDの管理画面②。「上位」と「下位」のリソースと、その対応関係が視覚的に理解できる。"deploy"はDeployment, "sts"はStatefulSet, "rs"はReplicaSetの略。

## 実際の運用

kubectl (CLI) を使い、黒いターミナル画面でカタカタやってるイメージがあるかもしれませんが、わざわざターミナル画面で苦しむことはありません。

- **リッチなGUI/自動化ツール**を用いて管理・監視を行える。
    - 例: Rancher, ArgoCD, Portainer, Lens (k8s), ...
- 必要なものが詰まった**様々なdistribution**が用意されているため、セットアップも楽になっている。
    - 例: k3s, k0s, microk8s, k3d, kind, ...

エコシステムがかなり整っているので、好きなGUIを使ったり、場合によってはコマンド一つでセットアップできるdistributionを使ったりできます。

### とにかくなんか触りたい人へ

とりあえず「触って理解したい！」な人には、コマンド一つで入るk3sや、環境を汚さず遊べるk3d(k3s on docker)が、個人的なオススメです。
NeoShowcaseの開発では、コマンド3ステップでk3dのインストール、環境の構築、環境の完全削除が可能になっています。[3ステップ構築](https://github.com/traPtitech/NeoShowcase/blob/main/docs/development.md#k8s-backend-k3d)

traPで運用しているArgoCDを覗いてみるのもオススメです。
※traP メンバー限定

https://cd.trap.jp/

## Furthuer Reading

公式ドキュメントを読もう！

- https://kubernetes.io/
- 用語集 https://kubernetes.io/docs/reference/glossary/?fundamental=true
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

## トリビア

kとsの間に8文字あるからk8sと略します。Numeronymというらしい。

Googleのインフラ管理システムがオープンソースになったプロジェクトです。
Golang製であり、現在も活発に開発が行われています。
GitHubのリポジトリの中で合計contributor数で数えると、Top 10に入るとか入らないとか。

## 次へ

Kubernetes の基本概念が理解できたら、ぜひハンズオンにトライしましょう！

[./1_tools.md](./1_tools.md)
