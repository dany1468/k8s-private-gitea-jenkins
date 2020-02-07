## 手順

### サービスアカウントの作成

ここでは `ci-sa` で作成するものとする。

### クラスタ作成

- ここでは限定公開クラスタで作成する。
- 以下ではマスターへのアクセスを制限していないが、必要に応じて行う。
- CIDR の設定についても必要な IP アドレスを計算した上で設定。
- NAT が無いとインターネットに出られないので適宜作成する。

```
$ gcloud container clusters create ci-cluster \
   --zone=asia-northeast1-b \
   --machine-type=g1-small \
   --disk-size=30 \
   --num-nodes=1 \
   --network=example \
   --subnetwork=public \
   --master-ipv4-cidr=172.16.0.32/28 \
   --default-max-pods-per-node=32 \
   --cluster-ipv4-cidr=/21 \
   --services-ipv4-cidr=/23 \
   --enable-ip-alias \
   --enable-private-nodes \
   --no-enable-master-authorized-networks # マスターへのアクセスを制限しない
   --service-account "ci-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com"
```

#### 参考

- [エイリアス IP アドレスを使用した VPC ネイティブ クラスタの作成](https://cloud.google.com/kubernetes-engine/docs/how-to/alias-ips?hl=ja)
- [IP アドレス割り当ての最適化](https://cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr?hl=ja)
- [限定公開クラスタの作成](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters/?hl=ja#all_access)
- [マスター アクセス用のネットワーク プロキシを持つ GKE 限定公開クラスタの作成](https://cloud.google.com/solutions/creating-kubernetes-engine-private-clusters-with-net-proxies?hl=ja)
- [GKEクラスタに割り当てるCIDRを設計する](https://future-architect.github.io/articles/20191017/)
- [GKEの限定公開クラスタに対するkubernetes APIのアクセス制御とCloudShellからのアクセス](https://scrapbox.io/spring-mt/GKE%E3%81%AE%E9%99%90%E5%AE%9A%E5%85%AC%E9%96%8B%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E3%81%AB%E5%AF%BE%E3%81%99%E3%82%8Bkubernetes_API%E3%81%AE%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E5%88%B6%E5%BE%A1%E3%81%A8CloudShell%E3%81%8B%E3%82%89%E3%81%AE%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9)
- [Cloud NATでGKEの外部IPを固定する](https://qiita.com/k_myoda/items/2100f66167ba11fcdcf3)
- [コントロール プレーンとノードへのネットワーク アクセスを制限する](https://cloud.google.com/kubernetes-engine/docs/how-to/hardening-your-cluster?hl=ja)

### 接続

```
$ gcloud container clusters get-credentials dev-cluster --zone asia-northeast1-b
$ kubectl config get-contexts
```

### サービスアカウントの key を secret に登録

ci-sa.json はサービスアカウントの JSON 形式のキー。

```
$ kubectl create secret generic ci-sa --from-file ci-sa.json
```

### Dockerfile のビルド

今回は以下の Dockerfile を使う

- docker/gitea-with-cloud-sdk/Dockerfile
  - Gitea 内で Cloud Source Repositories への mirroring を行うために `gcloud` コマンドを使うため
- docker/jenkins-with-docker/Dockerfile
  - Jenkins の Docker plugin を使うために、docker client を Jenkins のイメージに含める

```
$ docker build -t gitea-with-cloud-sdk -f docker/gitea-with-cloud-sdk/Dockerfile .
$ docker tag gitea-with-cloud-sdk gcr.io/$PROJECT_ID/gitea-with-cloud-sdk:x.x.x # 適宜バージョン
$ docker push gcr.io/$PROJECT_ID/gitea-with-cloud-sdk:x.x.x
```

上記で Container Registry に登録できたら ci-deployment.yaml の image 名も書き換える。

### deploy

```
$ kubectl apply -f k8s/
```

### gcloud の初期設定

```
$ export CI_POD=$(kubectl get pods -l "app=ci" -o jsonpath="{.items[0].metadata.name}")
$ kubectl exec -it $CI_POD --container gitea /bin/bash
```

以下はコンテナ内。

```
$ su - git
$ gcloud auth activate-service-account --key-file /data/key.json
```

一度実行すれば `/data/git/.config` に設定が書き込まれ、永続ディスクに保存されるため、Pod を再起動等しても問題無い。

## 接続

```
$ kubectl port-forward $GITEA_POD 9000
```

- http://localhost:9000/gitea
- http://localhost:9000/jenkins
