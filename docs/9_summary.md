# まとめ

ハンズオンと関連する概念の勉強、たいへんお疲れ様でした。

これで基本的なアプリケーションは、Kubernetes を用いてデプロイできるようになったはずです。

これ以外にも Kubernetes に関して学ぶことは非常に多く、ユーザーも多いため、エコシステムは日々進化し続けています。
しかし、今回のハンズオンで「基本的なアプリケーションがデプロイできるようになった」ことからも分かるように、Kubernetes エコシステムの全てを理解していなければ Kubernetes の利便性を享受できない、なんてことはありません。

むしろ、Kubernetes の利便性を享受し始めるのに必要な知識は、エコシステム全体に比べれば本当に少ないのです。
今回のハンズオンでは、この知識を網羅することを目指しました。

こうした最低限の知識は、公式ドキュメント[^1]にもよくまとまっています。
日本語のページを開くと分かりますが、英語版の翻訳が追いついているわけではない or 翻訳が完璧でないことも多々あるので、少し頑張ってでもオリジナルの英語版ドキュメントを読むことをおすすめします。
鬼畜な大学受験の英語みたいに難しい単語があるわけでもなく、むしろネイティブではない人でも読めるように、簡単な単語が使われていることがほとんどです。

[^1]: https://kubernetes.io/docs/home/

## And Keep Learning ...

このページのこれ以降では、ハンズオンの途中で扱うには少し難しい概念を、いくつか紹介して終わりとします。
これら全てを、今すぐ覚えておく必要はありません。
ぜひ「こんな名前の概念があったな」という「知識のポインター」だけを頭の中に入れておいて、それぞれ必要になった時に読んでみてください。

※私の知識も有限なので、網羅できているとは限りません。
ぜひご自分でも最新の動向を探ってみてください。

> [!NOTE]
> このハンズオン資料に typo などがあれば、こっそり Pull Request を送って教えてください！
> ここまで読んでいただき、ありがとうございました。

----

## Kubernetes 用語集

Kubernetes エコシステムで使われる用語集です。
ハンズオンで説明しきれなかった用語もあるので、覗いてみてください。
暇な時に読むも良し、辞書的に使うも良しです。

https://kubernetes.io/docs/reference/glossary/?fundamental=true

## リソース管理

Kubernetes の目的の一つとして、大量のトラフィックを捌くことがあります。
そのために大量のコンテナ（Pod）を、大量のノードに適切に均等に配置しなければいけません。

各コンテナにはある程度 CPU やメモリを必要とし、各ノードにも CPU やメモリリソースの上限があります。
各コンテナ（Pod）を最適にノードに配置する問題は、[binpacking problem](https://en.wikipedia.org/wiki/Bin_packing_problem) と同等です。

各 Pod で「必要な CPU やメモリのリソース量」を宣言することにより、Kubernetes は自動的にこの binpacking problem を解決し、最適なノードに Pod を配置してくれます。

https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

## HPA (Horizontal Pod Autoscaler) / Cluster Autoscaler

上のリソース管理と似た話です。

Kubernetes クラスターでデプロイされているWebサイトやアプリケーションへのトラフィック量は、ユーザーの上限、時間帯などによって大きく上下します。
したがって、Deployment などの replica 数や、それをサポートするノード数も動的に調整できると、クラウドではコストを最適化できて嬉しいです。

負荷の量によって動的に Deployment などの replica 数を上限させるオブジェクトが、Horizontal Pod Autoscaler です。

https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

また、Pod 数が動的に増えることで、ノードのリソースが足りなくなることがあります。
この時、Cluster Autoscaler を活用することにより、動的にクラスターのノード数を上限させることができます。

https://github.com/kubernetes/autoscaler

## CRI (Container Runtime Interface)

Kubernetes がコンテナを動かす際に利用する、プラグインの interface（仕様セット）です。

https://kubernetes.io/docs/concepts/architecture/cri/

Kubernetes がコンテナを動かすバックエンドとして、

- containerd
- CRI-O
- Docker Engine (deprecated)

などがあります。
今回のハンズオンで使った k3s は、containerd を使っています。

Docker も containerd をコンテナを動かすバックエンドとして使うようなので、結局の依存関係は似たものがありますね。

- Kubernetes → containerd → runc
- Docker Engine → containerd → runc

## OCI (Open Container Initiative)

近年複数の企業が集まってできた、コンテナ関連の標準化機構です。

https://opencontainers.org/

高レベルのコンテナランタイムである containerd などと、低レベルのコンテナランタイムである runc などの間の通信方式を「OCI Runtime Spec」が定義しています。
標準化という意味で、上の CRI と似ていますね。

また、「OCI Image and Distribution Specs」によって、いわゆる Docker Image と Docker Registry の形式を標準化しています。
これに（だいたい）従っている Image や Registry がほとんどであり、Kubernetes から「いつもの」Docker Image や Docker Hub をそのまま使えるのも、この標準化のおかげです。

Docker Engine を使ってイメージを build、push したときも（だいたい）この標準形式に従っています。

## クラウドとオンプレミスクラスターの違い

ハンズオンの各所でも紹介しましたが、Kubernetes にはそれ自身で使える要素と、「プロバイダー依存」の要素があります。

例:

- `type: LoadBalancer` Service
- Ingress
- PersistentVolume(Claim)
- Cluster Autoscaler
    - Pod の需要によって動的にノードをプロビジョニングする。

Kubernetes の利便性や耐障害性、高可用性といった性質は、もちろん Kubernetes 自身のアーキテクチャによって実現される部分も大きいですが、こうした「プロバイダー依存」のコンポーネントによって実現されている部分もあります。

クラウドは物理的な計算リソースを上手く柔らかく扱いやすい形で抽象化しており、これをさらに wrap して使いやすくしているのが Kubernetes の「プロバイダー依存」のコンポーネントとも捉えられます。

各クラウドとそれに対応する Kubernetes distribution を使う場合は、これらのコンポーネントを便利に享受できますが、
これらはあくまで「プロバイダー依存」であることを胸に留めておきたいところです。
過度に独自の便利な機能に依存してしまうと、ベンダーロックイン（簡単に別のプロバイダーにシステムを移行できないこと）してしまうことには注意が必要です。

裏の全ての仕組みを理解する必要はありませんが、どこまでがそのクラウドの機能なのかに目を向けてみると理解が深まり、より適切な利用法に繋がるでしょう。

## オブジェクト管理と GitOps

このハンズオンでは終始 kubectl CLI を用いて、クラスターの状態を操作しました。

小規模かつ、あまり変更が無い場合はこれでも十分かもしれません。
しかしより大規模になったり、オブジェクトに変更が頻繁に起きる場合は、手動で kubectl CLI を叩くことによるヒューマンエラーを削減したいです。

CI/CD を用いて、ある git リポジトリのイベントをトリガーとして `kubectl apply` などの自動化を行っても良いですが、より良い"ドリフト"が無いことを保証する方法があります。

まず Kubernetes は、その reconciliation loop によって「API Server に登録されているオブジェクト <-> 実際のコンテナ等」に乖離が無いことを機械的に保証するシステムです。

**GitOps**は、reconciliation loop によって「Git のオブジェクト yaml 定義 <-> Kubernetes API Server に登録されているオブジェクト」に乖離が無いことを保証するシステムです。

GitOps を用いることにより、開発者は git リポジトリに対して変更を行うだけとなり、複数人のコラボレーションやバージョン管理、問題発生時の切り戻しがさらに容易になります。

有名な GitOps OSS として、次のものがあります。

- [ArgoCD](https://github.com/argoproj/argo-cd)
- [Flux](https://github.com/fluxcd)

特に ArgoCD は分かりやすい UI がデフォルトで存在し、オブジェクトの依存関係などを視覚的に理解できるため、個人的にもお気に入りです。

## CSI (Container Storage Interface)

CSI は、Kubernetes にプラグイン形式で各種ストレージソリューションに対応する StorageClass を追加できる標準機構です。

各種クラウドの動的なブロックボリュームのアタッチなどを、CSI を通して行えます。

https://kubernetes.io/docs/reference/glossary/?storage=true#term-csi

昔は [Kubernetes](https://github.com/kubernetes/kubernetes) 自身のソースコードに、有名どころのクラウドの StorageClass 実装が入っていました（"in-tree" provider と呼びます）。
しかしその数が増えるにつれてソースコードやバイナリサイズも肥大化してしまうため、CSI という標準機構が作られ、全てのプロバイダーは徐々にそちらに移行しつつあります（"out-of-tree" plugin と呼びます）。

## Service Mesh

Kubernetes などでたくさんの種類のアプリケーションをデプロイしていると、通信方式の統一（プロトコルや暗号化するか等）やリトライ処理、Observability の確保などの、通信処理の統一が負担となってきます。

ここで各 Pod にプロキシーコンテナを同居させ、各アプリケーションは他のアプリケーションへ直接ではなく、そのプロキシーコンテナ宛に通信します。
Service Meshはこのように、「通信処理をアプリケーションから分離」し一括で管理してしまうという方法です。

このプロキシーを通して通信することで、さらに通信の制御が柔軟になり、複数の Kubernetes クラスター間で通信を行うこともあります。

https://dev.classmethod.jp/articles/servicemesh/

## 最近の動向

最近（2024年5月現在）で動きがある要素をいくつか紹介します。

その他にも、Kubernetes Blog を漁ると面白い動向が発見できるかもしれません。

https://kubernetes.io/blog/

### Gateway API

Ingress に取って代わる、より高機能な L4 / L7 ルーティングオブジェクトの一覧です。

https://gateway-api.sigs.k8s.io/

Ingress には足りなかった開発者のロールを意識したオブジェクト設計を取り入れたり、より複雑なルーティングを表現できるようになっています。

### Dynamic Resource Allocation

PV と PVC および CSI では、コンテナにファイルシステムのストレージというリソースを、動的に提供しました。

しかし、これ以外にもコンテナに違った種類のリソースを提供したいことがあるかもしれません。
任意の種類のリソースの動的な提供を、Kubernetes を通じて可能にしようという動きです。

https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/
