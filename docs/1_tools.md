# ツールの準備

以下のツールを用意します。
既に入っている場合はスキップします。

メジャーな使い方しかしないため、バージョン違いは大きな問題にならないと思います。
それぞれ最新バージョンをインストールしていただいて結構です。
一応、検証時のバージョンも併記しておきます。

## WSL2 (Windowsの場合)

目的:
Windows の場合、WSL2 を使うことで Linux 環境にかなり近づきます。
Docker や Kubernetes などのコンテナ環境は、主に Linux 環境を想定しています。
これらを試す際は Linux 環境で行うことを強くおすすめします。

方法:

- [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) (Ubuntu推奨) をインストールします。

以下、Windows を使っている方は、WSL2 上で操作を行い、「Linux」の指示に従ってハンズオンを進めてください。

## Docker

目的:
後ほどインストールする k3d で、Docker コンテナ上にクラスターを構築します。
こうすることで、ハンズオンの後片付けがとても簡単になります。

方法:

- macOS: [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/) をインストール
- Linux: [Docker Engine](https://docs.docker.com/engine/install/ubuntu/) をインストール

検証時: Docker Engine - Community 26.1.2

## kubectl

目的:
Kubernetes クラスターを操作する、最も原始的な方法です。

方法:

- macOS: [指示](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/) に従ってインストール
```shell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
sudo chown root: /usr/local/bin/kubectl
kubectl version --client # 確認
```

- Linux: [指示](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) に従ってインストール
```shell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client # 確認
```

検証時: Client Version v1.30.1

## k3d

目的:
Docker コンテナ上に Kubernetes クラスターを構築するツールです。
後片付けを簡単にし、手元の環境をほぼ汚さずにハンズオンを行うことができます。

方法:

- macOS, Windows: `curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash`
    - [k3d ドキュメント](https://k3d.io/)

検証時: v5.6.3

## 次へ

準備ができたら、早速触ってみましょう！

[./2_first_cluster.md](./2_first_cluster.md)
