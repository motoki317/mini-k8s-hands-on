# Volumes

前ページでは、ConfigMap や Secret からコンテナの Volume Mount を構成する方法を軽く学びました。

これ以外にも、一時的なボリュームや、永続的なボリュームをコンテナに Mount する様々な方法[^1]があります。

[^1]: https://kubernetes.io/docs/concepts/storage/volumes/

## 一時ボリューム

ConfigMap や Secret 以外にも、**一時的な**ボリュームをコンテナに Mount する方法があります。

「一時的な」とは、Pod が削除された後は Volume の内容は一切消えてしまう、という意味です。
したがって、コンテナの再起動・Pod 再作成間で保持したいデータを保存するには向きません。

### emptyDir

Volume 定義で `emptyDir` を使い、Pod が存在する間有効な、空のディレクトリーの Volume Mount を構成できます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-vol-example

spec:
  volumes:
    # 空のディレクトリーの、一時 Volume を構成
    - name: my-shared-vol
      emptyDir: {}

  containers:
    - name: demo-1
      image: demo:latest
      volumeMounts:
        - mountPath: /mount-point-1
          name: my-shared-vol

    - name: demo-2
      image: demo:latest
      volumeMounts:
        - mountPath: /mount-point-2
          name: my-shared-vol
```

Pod は**複数コンテナ**から構成可能なことを思い出し、Volume はコンテナではなく **Pod 単位で紐づく**ことに注目してください。

これを利用すると、上の Pod は `demo-1` コンテナと `demo-2` コンテナで一つの Volume を共有できます。
つまり、`demo-1` コンテナが `/mount-point-1/test.txt` というファイルを書き出した場合、
`demo-2` コンテナは `/mount-point-2/test.txt` としてこのファイルを読み出すことができます。

[NeoShowcase の builder Pod](https://github.com/traPtitech/manifest/blob/03ce70240d5e309777a57ac8820a8adf22d07f6f/ns-system/components/builder-deployment.yaml) では、emptyDir を使用して、UNIX domain socket ファイルの共有をコンテナ間で行っています。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ns-builder

spec:
  selector:
    matchLabels:
      app: ns-builder

  template:
    metadata:
      labels:
        app: ns-builder

    spec:
      volumes:
        - name: socket
          emptyDir: {}

      containers:
        - name: buildkitd
          image: moby/buildkit:v0.13.2-rootless
          volumeMounts:
            - mountPath: /run/user/1000/buildkit
              name: socket

        - name: builder
          image: ns-builder
          volumeMounts:
            - mountPath: /run/buildkit
              name: socket
```

上の例では、`buildkitd` コンテナが書き出した `/run/user/1000/buildkit/buildkitd.sock` ファイルを、
`builder` コンテナが `/run/buildkit/buildkitd.sock` として読み出し、UNIX domain socket に接続します。

## 永続ボリューム

データベースなどのステートフルなコンテナを Kubernetes 上にデプロイしたいとします。
このとき、コンテナ再起動・Pod 再作成間で特定のファイルを失いたくない（**永続化したい**）場合があります。

こうした場合に、**永続ボリューム**を使います。

### hostPath

Pod がデプロイされる Worker node のファイルシステムのパスを、そのまま Pod 上に Volume Mount できます。

Docker や Docker Compose で使い慣れた人も居るであろう、[Bind mount](https://docs.docker.com/storage/bind-mounts/) と似た機能です。

hostPath Volume を用いる時は、Pod を特定の Worker node のみでデプロイしたいことがあるため、そのような時は Pod 定義の `.spec.nodeSelector` も同時に活用します。

例えば、`mariadb` コンテナを `k3d-hands-on-agent-0` ノードにデプロイし、ホストの `/data/mariadb` パスを Volume Mount したい時は次のような定義になります。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mariadb

spec:
  # kubernetes.io/hostname ラベルが k3d-hands-on-agent-0 であるノードにのみ、Pod をデプロイする
  nodeSelector:
    kubernetes.io/hostname: k3d-hands-on-agent-0

  volumes:
    - name: data
      # ホストの /data/mariadb を Volume として定義
      hostPath:
        path: /data/mariadb
        type: DirectoryOrCreate

  containers:
    - name: mariadb
      image: mariadb:11
      env:
        - name: MARIADB_ROOT_PASSWORD
          value: password
      volumeMounts:
        # コンテナ内の /var/lib/mysql に Volume Mount
        - mountPath: /var/lib/mysql
          name: data
```

[../examples/7_hostpath_pod.yaml](../examples/7_hostpath_pod.yaml)

> [!NOTE]
> 再びですが、通常は Pod を直接定義せず、StatefulSet などを使います。

新しいフィールドを解説します。

- `.spec.nodeSelector` : デプロイするノードのセレクターを定義します。セレクターは label と value 形式で記述し、これに一致するノードにのみ Pod がスケジュールされます。
- `.spec.volumes[].hostPath.path` : ホスト上の Volume として使用するファイルシステムのパスを指定します。

> [!NOTE]
> ボーナス: ノードの label 一覧を確認し、`kubernetes.io/hostname` という label があることを確認してみましょう。
>
> - `kubectl get node k3d-hands-on-agent-0 -o yaml`

実際にこの Pod をデプロイし、Pod の再作成でデータが失われないことを確認しましょう。

次のコマンドで Pod をクラスターに登録します。

- `kubectl apply -f ./examples/7_hostpath_pod.yaml`

次のコマンドで Pod の 3306 番ポートに port-forward します。

- `kubectl port-forward pod/mariadb 3306:3306`

別ターミナルから、MySQL Client を使って接続します。

- `docker run --rm -it --network host --entrypoint sh mysql:8`
    - 手元に MySQL Client がある場合は持ってこなくても良いです。

次のコマンドでデータベースに接続します。

- `mysql -h127.0.0.1 -uroot -ppassword`

```plaintext
# mysql -h127.0.0.1 -uroot -ppassword
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 11.3.2-MariaDB-1:11.3.2+maria~ubu2204 mariadb.org binary distribution

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE DATABASE hands_on;
Query OK, 1 row affected (0.00 sec)

mysql> USE hands_on
Database changed
mysql> CREATE TABLE test (a INT);
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO test VALUES (1), (2), (3);
Query OK, 3 rows affected (0.01 sec)
Records: 3  Duplicates: 0  Warnings: 0

mysql> SELECT * FROM test;
+------+
| a    |
+------+
|    1 |
|    2 |
|    3 |
+------+
3 rows in set (0.00 sec)

mysql> ^DBye # CTRL + D で接続を切る
```

適当なデータ操作をした後、CTRL + D で接続を切ります。

さて、Pod を再作成してもデータが保持されるかを確かめましょう。

次のコマンドで Pod を削除し、再作成します。

- `kubectl delete -f ./examples/7_hostpath_pod.yaml`
- `kubectl get pods`
    - Pod が無くなることを確認します。
- `kubectl apply -f ./examples/7_hostpath_pod.yaml`

再度 port-forward を行い、別ターミナルからデータベースに接続しましょう。

- `kubectl port-forward pod/mariadb 3306:3306`
- `mysql -h127.0.0.1 -uroot -ppassword`

```plaintext
# mysql -h127.0.0.1 -uroot -ppassword
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 11.3.2-MariaDB-1:11.3.2+maria~ubu2204 mariadb.org binary distribution

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> USE hands_on;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> SELECT * FROM test;
+------+
| a    |
+------+
|    1 |
|    2 |
|    3 |
+------+
3 rows in set (0.00 sec)
```

先ほどのデータ操作の痕跡が確認できたでしょうか？

Pod 再作成間でデータの永続化ができました。
これで、**指定したノードで障害が起きない限り**、データの永続化が可能となります。

確認ができたら、port-forward を終了し、Pod を削除しておきましょう。

- `kubectl delete pod mariadb`

### PersistentVolume (PV) / PersistentVolumeClaim (PVC)

PersistentVolume[^2] は、Volume をより抽象化したリソースです。

[^2]: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

PersistentVolume（以下、PV）リソースは、ディスクなどの Volume 資源を表します。
PersistentVolumeClaim（以下、PVC）リソースを使って、PV（の一部）を Volume として確保できます。
PVC によって確保された Volume を、Pod へ Volume Mount できます。

実際の記述例を見てみましょう。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb

spec:
  storageClassName: standard-rwo
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi

---
apiVersion: v1
kind: Pod
metadata:
  name: mariadb

spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: mariadb

  containers:
    - name: mariadb
      image: mariadb:11
      env:
        - name: MARIADB_ROOT_PASSWORD
          value: password
      volumeMounts:
        - mountPath: /var/lib/mysql
          name: data
```

PVC の定義を少し解説します。

- `.spec.storageClassName` : StorageClass（後述）の名前。PV の「実装」を選択できる。
- `.spec.accessModes` : Volume が同時に複数ノードから読み書きできるかを指定する。`ReadWriteOnce` は、一つのノードからのみ同時に利用可能なことを表す。
- `.spec.resources.requests.storage` : PVC の大きさを指定する。

#### PV / PVC のプロバイダー依存性

StorageClass という単語が出てきました。

PV / PVC も、実は「プロバイダー依存」のリソースです。
StorageClass リソースは、特定の「プロバイダー実装」の PV / PVC を表します。
（Ingress と IngressClass を思い出すと、似た概念であることがわかります。）

先ほどは hostPath で Worker node のパスを直接 Volume Mount しました。
しかしこれでは Pod の配置先 Worker node が固定されてしまう上に、ノードに障害が起こると Pod が配置できなくなり、データにもアクセスできなくなります。

この問題を解決するのが PV と PVC 及び、そのプロバイダー実装です。
クラウド（例: GKE, EKS, AKS）による StorageClass では、PV の実体はブロックボリュームであったりします。
このブロックボリュームである PV を、我々が PVC 経由で Pod から Volume Mount して使うことで、StorageClass 実装は Pod が配置される Worker node にブロックボリュームを動的にアタッチします。

したがってノードに障害が起きたとしても、多くのクラウドではブロックボリュームは別のリソースとして確保されているため、ボリュームにまで影響は及びません。
別ノードにブロックボリュームを再アタッチし、Pod をそのノードにデプロイし直すことができます。

また、PV を手動で定義することもありますが、通常は StorageClass 実装を通じて、PVC から動的に PV が作られます[^3]。

[^3]: https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes?hl=ja#dynamic_provisioning

#### クラウド以外での PV / PVC

PV / PVC はプロバイダー依存のリソースであり、その安定性・耐障害性はクラウドの StorageClass 実装に依るところが大きいです。

したがって、オンプレミスの Kubernetes クラスターや、今回のハンズオンのように手元で動かしているクラスターでは、こうした耐障害性の高い StorageClass 実装をすぐには利用できません。
自身で StorageClass 実装を提供する必要があります。
（正確には、StorageClass と、それに紐づく provisioner 実装を提供する必要があります。）

こうした環境で利用できる StorageClass 実装では、次のようなものがあります。

- [Rook](https://rook.io/) : Ceph を使うことで分散ストレージシステムを使える。
- [yandex-cloud/k8s-csi-s3](https://github.com/yandex-cloud/k8s-csi-s3) : S3-compatible なストレージをバックエンドとして、FuseFS としてマウントする。パフォーマンスは見込めないが、少量の PVC に有効。
- [rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner) : Worker node のローカルのストレージを、動的に PV / PVC として使えるようにする。耐障害性では hostPath と同じ。

今回のハンズオンでは、k3s のデフォルトである local-path-provisioner が使えます。

次のコマンドで StorageClass を確認しましょう。

- `kubectl get storageclass`

`local-path` という名前の StorageClass があれば正解です。

```plaintext
$ kubectl get storageclass
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  119m
```

#### PV / PVC を使う

概念的には hostPath とほとんど同じですが、この `local-path` StorageClass を使ってみましょう。

まずは PVC リソースを作成します。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb

spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

[../examples/7_pvc.yaml](../examples/7_pvc.yaml)

- `kubectl apply -f ./examples/7_pvc.yaml`

無事作成されたかを確認します。

- `kubectl get pvc`
    - `persistentvolumeclaim` の略として、`pvc` と書けます。他にも `service` の略は `svc` など、リソースには略称が存在することがあります。

```plaintext
$ kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mariadb   Pending                                      local-path     107s
```

`STATUS: Pending` となっていますが、これで正常です。
なぜ `Pending` となっているのでしょうか？
リソースの詳細（イベント）を確認してみましょう。

- `kubectl describe pvc mariadb`

```plaintext
$ kubectl describe pvc mariadb
Name:          mariadb
Namespace:     default
StorageClass:  local-path
Status:        Pending
Volume:
Labels:        <none>
Annotations:   <none>
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:
Access Modes:
VolumeMode:    Filesystem
Used By:       <none>
Events:
  Type    Reason                Age                   From                         Message
  ----    ------                ----                  ----                         -------
  Normal  WaitForFirstConsumer  10s (x11 over 2m35s)  persistentvolume-controller  waiting for first consumer to be created before binding
```

`WaitForFirstConsumer` というイベントが記録されています。
文字通り、PVC を最初に使う（「消費する」）Pod が現れない限り、実際にはボリュームは確保されません。
この挙動は、StorageClass の設定や実装に依ります。

次に、この PVC を Volume Mount する Pod を作成しましょう。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mariadb

spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: mariadb

  containers:
    - name: mariadb
      image: mariadb:11
      env:
        - name: MARIADB_ROOT_PASSWORD
          value: password
      volumeMounts:
        - mountPath: /var/lib/mysql
          name: data
```

[../examples/7_pvc_pod.yaml](../examples/7_pvc_pod.yaml)

- `kubectl apply -f ./examples/7_pvc_pod.yaml`

Pod と PVC の状態を確認しましょう。

- `kubectl get pods`
- `kubectl get pvc`

```plaintext
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
mariadb   1/1     Running   0          31s

$ kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mariadb   Bound    pvc-6a18417b-eddc-499a-9450-ead0be9ed0c1   10Gi       RWO            local-path     6m50s
```

先ほどとは異なり、PVC リソースの `STATUS` が `Bound` となりました。
無事 PVC を使用できたようです。

> [!NOTE]
> ボーナス: それぞれのリソースのイベントを確認し、PVC がプロビジョニングされる様子を確認しましょう。
>
> - `kubectl describe pvc mariadb`
> - `kubectl describe pod mariadb`

最後に、Pod へ接続してデータ操作を行い、Pod を削除・再作成し、データが永続化されているかを確かめましょう。
ここでは PVC リソースに Volume データの実体が紐づいているため、**PVC リソースを消すと Volume データも消えます**ので、注意してください。

細かい手順は hostPath の時と同じなので省きますが、次の通りです。

1. port-forward 経由で、データベースに接続する
2. 適当なデータ操作を行う
3. Pod を削除し、再作成する
4. 再度データベースに接続し、データが残っていることを確認する

確認ができたら成功です。
Pod と PVC を削除しておきましょう。

- `kubectl delete pod mariadb`
- `kubectl delete pvc mariadb`

## 次へ

お疲れ様でした！

このページでは、Pod の一時・永続ボリュームについて詳しく学びました。
より詳しく知りたい場合は、以下の footnote などから、様々なドキュメントを漁ってみてください。

これで**本当に基本的な Kubernetes の要素**は一通り学びました。
次のページでは、演習として [traQ](https://github.com/traPtitech/traQ) のデプロイに挑戦します。

[./8_exercise.md](./8_exercise.md)
