---
title: gRPC/Protocol BuffersのAPIドキュメントを自動生成してGitHub Pagesに継続的デプロイする
emoji: "📖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "grpc", "protocolbuffers", "githubactions", "githubpages", "ci" ]
published: true
---

皆さんはgRPC/Protocol BuffersのAPI仕様書などをどのように管理されていますか？
Notionやesaなどのドキュメントツールに記載している方も多いと思いますが、バージョン管理されているprotoファイルとの整合性を保つのは大変ですよね。
今回は[protoc-gen-doc](https://github.com/pseudomuto/protoc-gen-doc)を使用してリポジトリ内のprotoファイルからAPIドキュメントを生成し、GitHub Pagesに継続的デプロイする方法を紹介します。

![Protocol Documentation](/images/grpc-proto-generate-document/document_300w.jpg)

## GitHub Pagesの設定変更

最初にGitHub Pagesの設定を変更し、GitHub Actionsからデプロイできるようにします。
リポジトリのSettingsからGitHub Pagesの設定を開き、Build and deploymentのSourceを **GitHub Actions** に変更します。

### アクセス制限について

* Enterpriseプランの場合はGitHub Pages visibilityからPrivateに変更することが出来ます。これにより該当のリポジトリへのアクセス権限があるユーザーのみがドキュメントを閲覧できるようになります。
* EnterpriseプランではないOrganizationの場合はアクセス制限をかけることが出来ませんが、GitHub PagesのURLがランダムで生成されるため、URLを知らない限りは基本的にはアクセスできないようになっています。(ただし確実ではないかも・・・)

![GitHub Pages Settings](/images/grpc-proto-generate-document/githubpages-source.jpg)

## ディレクトリ構成

今回は以下のディレクトリ構成で進めていきます。既存のプロジェクトに導入する際はprotoファイルのディレクトリ構成に合わせて変更してください。

```
.
├── .github
│   └── workflows
│       └── generate-document.yml # GitHub ActionsのWorkflow
├── api
│   └── proto
│       └── helloworld
│          └── helloworld.proto # protoファイル
└─　README.md
```

## GitHub ActionsのWorkflow用意

ドキュメント生成とデプロイを行うGitHub ActionsのWorkflowを用意します。
下記のWorkflowをリポジトリの`.github/workflows/generate-document.yml`に配置してください。

```yaml
name: Generate Document
on:
  # mainブランチへのpush時に実行。それぞれのブランチに合わせて変更します。
  push:
    branches:
      - main
  workflow_dispatch:

# 同時実行を許容しない
concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  generate:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    permissions:
      contents: read
      pages: write
      id-token: write
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Setup Pages
        uses: actions/configure-pages@v3
      - name: Run protoc-gen-doc
        run: |
          docker run --rm -v $(pwd)/out:/out -v $(pwd)/api/proto:/proto pseudomuto/protoc-gen-doc --proto_path=/proto --doc_opt=html,index.html $(find api/proto -name "*.proto" | sed "s/api\/proto\///g")
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: out
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2
```

## テスト用のprotoファイルを用意

ドキュメント生成のテストのためprotoファイルを `api/proto/helloworld/helloworld.proto` に配置します。

```proto
syntax = "proto3";

package helloworld.helloworld;

/**
 * The greeting service definition.
 */
service HelloWorldService {
  // Sends a greeting
  rpc SayHello(HelloRequest) returns (HelloResponse);
}

/**
 * The request message containing the user's name.
 */
message HelloRequest {
  string name = 1; // The name of the person to whom to say hello
}

/**
 * The response message containing the greetings
 */
message HelloResponse {
  string message = 1; // The greeting
}
```

## ドキュメント生成のテスト

これでprotoファイルとWorkflowが準備ができたので、ドキュメント生成のテストを行います。
リポジトリのmainブランチにpushすると、GitHub Actionsが実行され、ドキュメントが生成されます。

生成されたドキュメントのデプロイ先URLはActionsのSummaryまたはDeployment HistoryのView deploymentで確認できます。

![GitHub Actions Summary](/images/grpc-proto-generate-document/actions-summary.jpg)

![Deployment History - View deployment](/images/grpc-proto-generate-document/view-deployment.jpg)

## 最後に

今回はgRPC/Protocol Buffersのドキュメントを自動生成してGitHub Pagesにデプロイする方法を紹介しました。
protoファイルの変更を検知してドキュメントを自動更新することが出来るので、チーム内でAPI仕様書を共有する際に役立つかと思います。

* サンプルリポジトリ https://github.com/aplulu/protoc-gen-doc-github-pages-example
* 生成したドキュメント https://aplulu.github.io/protoc-gen-doc-github-pages-example/
