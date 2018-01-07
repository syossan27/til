# kubernetesを使ったデプロイ

CircleCI→DockerHub→GKEという流れでデプロイしてみるテスト。

## CircleCIセッティング

- Pojectの追加
- `.circleci/config.yml` を作成。サンプルは以下。

```
# Golang CircleCI 2.0 configuration file
#
# Check https://circleci.com/docs/2.0/language-go/ for more details
version: 2
jobs:
  build:
    docker:
      # specify the version
      - image: circleci/golang:1.8

      # Specify service dependencies here if necessary
      # CircleCI maintains a library of pre-built images
      # documented at https://circleci.com/docs/2.0/circleci-images/
      # - image: circleci/postgres:9.4

    #### TEMPLATE_NOTE: go expects specific checkout path representing url
    #### expecting it in the form of
    ####   /go/src/github.com/circleci/go-tool
    ####   /go/src/bitbucket.org/circleci/go-tool
    working_directory: /go/src/github.com/{{ORG_NAME}}/{{REPO_NAME}}
    steps:
      - checkout

      # specify any bash command here prefixed with `run: `
      - run: go get -v -t -d ./...
      - run: go test -v ./...
```

コンテナを起動させると `working_directory` にあるGoアプリがdaemonで動く。

## DockerHubセッティング

Automated buildを使い、Githubに特定のタグでpushされたらDockerHubにコンテナを登録する。  
↑  
これより、CircleCIでDockerHubにコンテナを送る。

### webhook

以下を参考に  
[DockerhubのAutomated build を設定する](https://qiita.com/FoxBoxsnet/items/a31fc46681d9b67c8cc6)

## 備考

- サンプルアプリを試す際にはdocker-machineのIPにアクセスして試す
- DockerHubのprivateでbuildしたコンテナをpullする場合は、docker loginしてね
