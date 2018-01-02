# kubernetes

## 構成

cluster：kubernetes上の最上位の概念。主にサービス単位などに分ける。
master：nodeを管理する。Kubernetes操作APIの提供や、node管理作業、nodeのコンテナ割当などを担う。GKEではあまり意識しない概念。
node：podを管理する。podを管理するエージェント・kubeletやkube-proxyが動いている。
pod：コンテナが動いている。
service：podと外部が通信できるようにする。

## 参考資料

- [kubernetes初心者のための入門ハンズオン](https://qiita.com/mihirat/items/ebb0833d50c882398b0f)
