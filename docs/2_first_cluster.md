# 最初のクラスター

## クラスターを立ち上げる

Kubernetes のインスタンスは「クラスター」と呼ばれます。
通常、複数の Linux ホスト（ノード）を管理するためです。

今回は、お手持ちの一つのマシンでハンズオンを行いたいため、
一つの docker コンテナを「一つの仮想的な Linux ホスト」として使っていきます。
これを可能にするツールの一つが、「k3d」です。

次のコマンドを実行しましょう。

- `k3d cluster create hands-on --agents 3 --k3s-arg "--disable=traefik@server:*"`
    - 細かい引数は後ほど解説します。そのままコピペしてください。

私の環境では30秒ほど掛かります。
次のような出力が出たら成功です。

```plaintext
$ k3d cluster create hands-on --agents 3 --k3s-arg "--disable=traefik@server:*"
INFO[0000] Prep: Network
INFO[0000] Created network 'k3d-hands-on'
INFO[0000] Created image volume k3d-hands-on-images
INFO[0000] Starting new tools node...
INFO[0001] Creating node 'k3d-hands-on-server-0'        
INFO[0001] Pulling image 'ghcr.io/k3d-io/k3d-tools:5.6.3' 
INFO[0003] Pulling image 'docker.io/rancher/k3s:v1.28.8-k3s1' 
INFO[0003] Starting node 'k3d-hands-on-tools'
INFO[0010] Creating node 'k3d-hands-on-agent-0'
INFO[0010] Creating node 'k3d-hands-on-agent-1'
INFO[0010] Creating node 'k3d-hands-on-agent-2'
INFO[0010] Creating LoadBalancer 'k3d-hands-on-serverlb' 
INFO[0011] Pulling image 'ghcr.io/k3d-io/k3d-proxy:5.6.3' 
INFO[0015] Using the k3d-tools node to gather environment information 
INFO[0015] HostIP: using network gateway 172.16.5.1 address 
INFO[0015] Starting cluster 'hands-on'
INFO[0015] Starting servers...
INFO[0015] Starting node 'k3d-hands-on-server-0'        
INFO[0018] Starting agents...
INFO[0018] Starting node 'k3d-hands-on-agent-1'
INFO[0018] Starting node 'k3d-hands-on-agent-0'
INFO[0018] Starting node 'k3d-hands-on-agent-2'
INFO[0023] Starting helpers...
INFO[0023] Starting node 'k3d-hands-on-serverlb'
INFO[0029] Injecting records for hostAliases (incl. host.k3d.internal) and for 5 network members into CoreDNS configmap...
INFO[0031] Cluster 'hands-on' created successfully!     
INFO[0031] You can now use it like this:
kubectl cluster-info
```

出力の最後でアドバイスがある通り、`kubectl cluster-info` も実行してみましょう。

次のような出力が出たら成功です。

```plaintext
$ kubectl cluster-info
Kubernetes control plane is running at https://0.0.0.0:38735
CoreDNS is running at https://0.0.0.0:38735/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:38735/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

クラスターに関する、基本的な情報を表示してくれます。
`0.0.0.0:38735`、つまりローカルにある「Kubernetes control plane」に接続していることが分かります。
（ポート番号は毎回異なるはずですが、気にしないで良いです。）

## ノードの確認

さて、先ほど「k3d」は、一つの docker コンテナを「一つの仮想的な Linux ホスト」として使うと書きました。
この状態では、docker コンテナがいくつか立ち上がっているはずです。

`docker ps` と打って、確認してみましょう。

```plaintext
$ docker ps
CONTAINER ID   IMAGE                            COMMAND                  CREATED         STATUS         PORTS                             NAMES
955026d6cad1   ghcr.io/k3d-io/k3d-proxy:5.6.3   "/bin/sh -c nginx-pr…"   5 minutes ago   Up 5 minutes   80/tcp, 0.0.0.0:38735->6443/tcp   k3d-hands-on-serverlb
63d93d25d2c1   rancher/k3s:v1.28.8-k3s1         "/bin/k3d-entrypoint…"   5 minutes ago   Up 5 minutes                
                     k3d-hands-on-agent-2
2c8ea7d6312b   rancher/k3s:v1.28.8-k3s1         "/bin/k3d-entrypoint…"   5 minutes ago   Up 5 minutes                
                     k3d-hands-on-agent-1
9893c7d63cb9   rancher/k3s:v1.28.8-k3s1         "/bin/k3d-entrypoint…"   5 minutes ago   Up 5 minutes                
                     k3d-hands-on-agent-0
b1a3e0de477b   rancher/k3s:v1.28.8-k3s1         "/bin/k3d-entrypoint…"   5 minutes ago   Up 5 minutes                
                     k3d-hands-on-server-0
```

次のコンテナが確認できると思います。

- 「server」のコンテナが一つ
    - 「Control plane」の役割をします。
- 「agent」のコンテナが三つ
    - 「Worker nodes」の役割をします。
- 「serverlb」のコンテナが一つ
    - k3d 独自の概念のため、あまり気にしないで良いです。手元のマシン（ホスト側）と docker コンテナの橋渡しをします。

`kubectl get nodes` でも確認してみましょう。

```shell
$ kubectl get nodes
NAME                    STATUS   ROLES                  AGE   VERSION
k3d-hands-on-agent-2    Ready    <none>                 11m   v1.28.8+k3s1
k3d-hands-on-server-0   Ready    control-plane,master   11m   v1.28.8+k3s1
k3d-hands-on-agent-1    Ready    <none>                 11m   v1.28.8+k3s1
k3d-hands-on-agent-0    Ready    <none>                 11m   v1.28.8+k3s1
```

serverlb 以外は、似た結果が得られるはずです。

Kubernetesでは、「司令塔」である「Control plane」と、実際にユーザーのコンテナ（アプリケーション）が動く「Worker nodes」は分かれています。

ここでは、2種類あることが確認できれば良いです。

![kubernetes components](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

Kubernetesクラスターを構成するコンポーネント[^1]

[^1]: https://kubernetes.io/ja/docs/concepts/overview/components/

## クラスターの終了

k3d を使うことで、いつでもハンズオンを終了できます。
ただし、**クラスターのデータは保存されません**。

次のコマンドで、クラスターを完全に削除します。

- `k3d cluster delete hands-on`

再開する時は、最初に行った cluster create コマンドで、再度クラスターを作成します。

本当に簡単なので、この時点でクラスターを一度削除して、再作成してみると面白いかもしれません。

## 次へ

クラスターの準備ができたら、いよいよコンテナをデプロイします。

[./3_first_containers.md](./3_first_containers.md)
