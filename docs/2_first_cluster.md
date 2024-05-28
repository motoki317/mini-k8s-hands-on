# 最初のクラスター

## クラスターを立ち上げる

Kubernetes のインスタンスは「クラスター」と呼ばれます。
通常、複数の Linux ホスト（ノード）を管理するためです。

今回は、お手持ちの一つのマシンでハンズオンを行いたいため、
「一つの docker コンテナ」を「一つの仮想的な Linux ホスト」として使います。
これを可能にするツールの一つが、「k3d」です。

次のコマンドを実行して、クラスターを立ち上げます。

- `k3d cluster create hands-on --agents 3 --port "80:80@loadbalancer" --port "8080:8080@loadbalancer"`
    - 細かい引数は後ほど解説します。そのままコピペしてください。

私の環境では30秒ほど掛かります。
次のような出力が出たら成功です。

```plaintext
$ k3d cluster create hands-on --agents 3 --port "80:80@loadbalancer" --port "8080:8080@loadbalancer"
INFO[0000] portmapping '80:80' targets the loadbalancer: defaulting to [servers:*:proxy agents:*:proxy]
INFO[0000] portmapping '8080:8080' targets the loadbalancer: defaulting to [servers:*:proxy agents:*:proxy]
INFO[0000] Prep: Network
INFO[0000] Created network 'k3d-hands-on'
INFO[0000] Created image volume k3d-hands-on-images     
INFO[0000] Starting new tools node...
INFO[0000] Starting node 'k3d-hands-on-tools'
INFO[0001] Creating node 'k3d-hands-on-server-0'        
INFO[0001] Creating node 'k3d-hands-on-agent-0'
INFO[0001] Creating node 'k3d-hands-on-agent-1'
INFO[0001] Creating node 'k3d-hands-on-agent-2'
INFO[0001] Creating LoadBalancer 'k3d-hands-on-serverlb' 
INFO[0001] Using the k3d-tools node to gather environment information 
INFO[0001] HostIP: using network gateway 172.16.7.1 address 
INFO[0001] Starting cluster 'hands-on'
INFO[0001] Starting servers...
INFO[0001] Starting node 'k3d-hands-on-server-0'        
INFO[0004] Starting agents...
INFO[0004] Starting node 'k3d-hands-on-agent-0'
INFO[0004] Starting node 'k3d-hands-on-agent-2'
INFO[0004] Starting node 'k3d-hands-on-agent-1'
INFO[0008] Starting helpers...
INFO[0008] Starting node 'k3d-hands-on-serverlb'
INFO[0014] Injecting records for hostAliases (incl. host.k3d.internal) and for 5 network members into CoreDNS configmap...
INFO[0016] Cluster 'hands-on' created successfully!     
INFO[0016] You can now use it like this:
kubectl cluster-info
```

出力の最後でアドバイスがある通り、`kubectl cluster-info` を実行してみましょう。

次のような出力が出たら成功です。

```plaintext
$ kubectl cluster-info
Kubernetes control plane is running at https://0.0.0.0:41407
CoreDNS is running at https://0.0.0.0:41407/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:41407/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

クラスターに関する、基本的な情報を表示してくれます。
`0.0.0.0:41407`、つまりローカルにある「Kubernetes control plane」に接続していることが分かります。
（ポート番号は毎回異なりますが、気にしないで良いです。）

## ノードの確認

さて先ほど、k3dは「一つの docker コンテナ」を「一つの仮想的な Linux ホスト」として使うと書きました。

クラスターを立ち上げたため、docker コンテナがいくつか立ち上がっているはずです。

`docker ps` と打って、確認します。

```plaintext
$ docker ps
CONTAINER ID   IMAGE                            COMMAND                  CREATED         STATUS         PORTS                                                                                                   NAMES
494f2c661714   ghcr.io/k3d-io/k3d-proxy:5.6.3   "/bin/sh -c nginx-pr…"   2 minutes ago   Up 2 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:41407->6443/tcp   k3d-hands-on-serverlb
48517abfd362   rancher/k3s:v1.28.8-k3s1         "/bin/k3d-entrypoint…"   2 minutes ago   Up 2 minutes                                                                                                           k3d-hands-on-agent-2
8b209ec273c4   rancher/k3s:v1.28.8-k3s1         "/bin/k3d-entrypoint…"   2 minutes ago   Up 2 minutes                                                                                                           k3d-hands-on-agent-1
e2606a53776b   rancher/k3s:v1.28.8-k3s1         "/bin/k3d-entrypoint…"   2 minutes ago   Up 2 minutes                                                                                                           k3d-hands-on-agent-0
c1c01542434e   rancher/k3s:v1.28.8-k3s1         "/bin/k3d-entrypoint…"   2 minutes ago   Up 2 minutes                                                                                                           k3d-hands-on-server-0
```

次のコンテナが確認できます。

- 「server」のコンテナが一つ
    - 「Control plane」の役割をします。
- 「agent」のコンテナが三つ
    - 「Worker nodes」の役割をします。
- 「serverlb」のコンテナが一つ
    - k3d 独自の概念のため、あまり気にしないで良いです。手元のマシン（ホスト側）と docker コンテナの橋渡しをします。

`kubectl get nodes` と打って、Kubernetes クラスターに認識されているノードを確認します。

```shell
$ kubectl get nodes
NAME                    STATUS   ROLES                  AGE   VERSION
k3d-hands-on-agent-2    Ready    <none>                 11m   v1.28.8+k3s1
k3d-hands-on-server-0   Ready    control-plane,master   11m   v1.28.8+k3s1
k3d-hands-on-agent-1    Ready    <none>                 11m   v1.28.8+k3s1
k3d-hands-on-agent-0    Ready    <none>                 11m   v1.28.8+k3s1
```

serverlb 以外は、`docker ps` と似た結果が得られます。

Kubernetes は、「司令塔」である「Control plane」と、実際にユーザーのコンテナ（アプリケーション）が動く「Worker nodes」に、大きく分かれています。

ここでは、この2種類があることが確認できれば良いです。

![kubernetes components](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

Kubernetesクラスターを構成するコンポーネント[^1]

[^1]: https://kubernetes.io/ja/docs/concepts/overview/components/

## クラスターの終了

k3d を使うことで、いつでもハンズオンを終了できます。
ただし、**クラスターの状態は保存されません**。

次のコマンドで、クラスターを完全に削除します。

- `k3d cluster delete hands-on`

再開する時は、最初に行った k3d cluster create コマンドで、再度クラスターを作成します。

本当に簡単なので、この時点でクラスターを一度削除して、再作成してみると面白いかもしれません。

## 次へ

クラスターの準備ができたら、いよいよコンテナをデプロイします。

[./3_first_containers.md](./3_first_containers.md)
