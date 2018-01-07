# kubernetes

## 構成

cluster：kubernetes上の最上位の概念。主にサービス単位などに分ける。  
master：nodeを管理する。Kubernetes操作APIの提供や、node管理作業、nodeのコンテナ割当などを担う。GKEではあまり意識しない概念。  
node：podを管理する。podを管理するエージェント・kubeletやkube-proxyが動いている。  
pod：コンテナが動いている。  
service：podと外部が通信できるようにする。  

Deployment: Replicasetを管理する。  
Replicaset: Podを管理する。  

## 単語

rolling update: サービスダウン無しに、podを順々にアップデートする。

## 覚書

- podを作成するとdeployment, replicasetも同時に作成される。

## コマンド

### クラスタ作成

```
gcloud container clusters create [クラスタ名]
```

## 参考資料

- [kubernetes初心者のための入門ハンズオン](https://qiita.com/mihirat/items/ebb0833d50c882398b0f)
- [Kubernetes Deploymentを理解する](https://qiita.com/komattaka/items/bd1d8d32f6cb24a32f53)
