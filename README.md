# ミニ k8s ハンズオン

これだけ知っておけば「だいたい Kubernetes を使える」になるまでを、（執筆者が思う）最短距離で触っていきます。
コマンドをコピペしていくだけでなく、それぞれにどんな意味があるのかに重点を置いて解説していきます。

想定する知識(必須ではありませんが、理解が深まります):

- コンテナの基礎知識
    - Docker や、そのコマンドである `docker build`, `docker run` を見た・触ったことがある
    - Dockerfile や Docker Image を見た・触ったことがある
- YAML の基本的な読み方
    - 連想配列と、リストだけ読めれば良いです。そんなに複雑な記法は使いません。

想定読者:

- Kubernetes を聞いたことがあるが、あまり触ったことがなく、やってみたい

## 仕組みから理解する

Kubernetes の基本概念を、短くまとめました。

ハンズオンを始める前に読んでおくと、理解が深まるかもしれません。

[./docs/0_kubernetes_workings.md](./docs/0_kubernetes_workings.md)

## ハンズオンから始める

触って理解する方が早いという人は、早速準備をしていきましょう！
次のページへ進んでください。

[./docs/1_tools.md](./docs/1_tools.md)
