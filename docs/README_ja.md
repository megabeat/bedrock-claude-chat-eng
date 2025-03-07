# Bedrock Claude Chat

> **Warning**
> 現在のバージョン(`v0.2.x`)は会話スキーマの変更により以前のバージョン(`v0.1.0`)との互換性がありません。以前のバージョンで DynamoDB に保存された会話はレンダリングできませんのでご注意ください。

このリポジトリは、生成系 AI を提供する[Amazon Bedrock](https://aws.amazon.com/jp/bedrock/)の基盤モデルの一つである、Anthropic 社製 LLM [Claude 2](https://www.anthropic.com/index/claude-2)を利用したチャットボットのサンプルです。

![](./imgs/demo2.gif)

## アーキテクチャ

AWS のマネージドサービスで構成した、インフラストラクチャ管理の不要なアーキテクチャとなっています。Amazon Bedrock の活用により、 AWS 外部の API と通信する必要がありません。スケーラブルで信頼性が高く、安全なアプリケーションをデプロイすることが可能です。

- [Amazon DynamoDB](https://aws.amazon.com/jp/dynamodb/): 会話履歴保存用の NoSQL データベース
- [Amazon API Gateway](https://aws.amazon.com/jp/api-gateway/) + [AWS Lambda](https://aws.amazon.com/jp/lambda/): バックエンド API エンドポイント ([AWS Lambda Web Adapter](https://github.com/awslabs/aws-lambda-web-adapter), [FastAPI](https://fastapi.tiangolo.com/))
- [Amazon SNS](https://aws.amazon.com/jp/sns/): API Gateway と Bedrock 間のストリーミング呼び出しを疎結合にするため使用しています。ストリーミングレスポンスにはトータルで 30 秒以上かかることがあり、これは HTTP インテグレーションの制約を超えてしまうためです（[クオータ](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html)を参照）。
- [Amazon CloudFront](https://aws.amazon.com/jp/cloudfront/) + [S3](https://aws.amazon.com/jp/s3/): フロントエンドアプリケーションの配信 ([React](https://react.dev/), [Tailwind CSS](https://tailwindcss.com/))
- [AWS WAF](https://aws.amazon.com/jp/waf/): IP アドレス制限
- [Amazon Cognito](https://aws.amazon.com/jp/cognito/): ユーザ認証
- [Amazon Bedrock](https://aws.amazon.com/jp/bedrock/): 基盤モデルを API 経由で利用できるマネージドサービス

![](imgs/arch.png)

## 機能・ロードマップ

- [x] 認証 (サインアップ・サインイン)
- [x] 会話の新規作成・保存・削除
- [x] チャットボットの返信内容のコピー
- [x] 会話の件名自動提案
- [x] コードのシンタックスハイライト
- [x] マークダウンのレンダリング
- [x] ストリーミングレスポンス
- [x] IP アドレス制限
- [x] メッセージの編集と再送
- [x] I18n 対応 (日/英)
- [ ] プロンプトテンプレートの保存と再利用

## プロジェクトのデプロイ

### 🚀 Easy Deployment

> Note: 2023/10 現在、Bedrock はすべてのリージョンをサポートしていません。以下の手順では Bedrock リソースを`us-east-1`にデプロイします（他のリソースは CloudShell が実行されたリージョンにデプロイされます）。Bedrock のリージョンを変更する必要がある場合は、この章の後の指示に従って CDK を直接使用してデプロイしてください。

- [CloudShell](https://console.aws.amazon.com/cloudshell/home)を開きます
- 下記のコマンドでリポジトリをクローンします

```sh
git clone https://github.com/aws-samples/bedrock-claude-chat.git
```

- 下記のコマンドでデプロイ実行します

```sh
cd bedrock-claude-chat
chmod +x bin.sh
./bin.sh
```

- 10 分ほど経過後、下記の出力が得られるのでブラウザからアクセスします

```
Frontend URL: https://xxxxxxxxx.cloudfront.net
```

![](./imgs/signin.png)

上記のようなサインアップ画面が現れますので、E メールを登録・ログインしご利用ください。

### Deploy using CDK

上記 Easy Deployment は[AWS CodeBuild](https://aws.amazon.com/jp/codebuild/)を利用し、内部で CDK によるデプロイを実行しています。ここでは直接 CDK によりデプロイする手順を記載します。

- お手元に UNIX コマンドおよび Node.js 実行環境を用意してください。もし無い場合、[Cloud9](https://github.com/aws-samples/cloud9-setup-for-prototyping)をご利用いただくことも可能です

- このリポジトリをクローンします

```
git clone https://github.com/aws-samples/bedrock-claude-chat
```

- npm パッケージをインストールします

```
cd bedrock-claude-chat
cd cdk
npm ci
```

- [AWS CDK](https://aws.amazon.com/jp/cdk/)をインストールします

```
npm i -g aws-cdk
```

- CDK デプロイ前に、デプロイ先リージョンに対して 1 度だけ Bootstrap の作業が必要となります。ここでは東京リージョンへデプロイするものとします。なお`<account id>`はアカウント ID に置換してください。

```
cdk bootstrap aws://<account id>/ap-northeast-1
```

- 必要に応じて[cdk.json](../cdk/cdk.json)の下記項目を編集します

  - `bedrockRegion`: Bedrock が利用できるリージョン
  - `allowedIpV4AddressRanges`, `allowedIpV6AddressRanges`: 許可する IP アドレス範囲の指定

- プロジェクトをデプロイします

```
cdk deploy --require-approval never --all
```

- 下記のような出力が得られれば成功です。`BedrockChatStack.FrontendURL`に WEB アプリの URL が出力されますので、ブラウザからアクセスしてください。

```sh
 ✅  BedrockChatStack

✨  Deployment time: 78.57s

Outputs:
BedrockChatStack.AuthUserPoolClientIdXXXXX = xxxxxxx
BedrockChatStack.AuthUserPoolIdXXXXXX = ap-northeast-1_XXXX
BedrockChatStack.BackendApiBackendApiUrlXXXXX = https://xxxxx.execute-api.ap-northeast-1.amazonaws.com
BedrockChatStack.FrontendURL = https://xxxxx.cloudfront.net
```

## その他

### テキスト生成パラメータの設定

[config.py](../backend/app/config.py)を編集後、`cdk deploy`を実行してください。

```py
GENERATION_CONFIG = {
    "max_tokens_to_sample": 500,
    "temperature": 0.6,
    "top_k": 250,
    "top_p": 0.999,
    "stop_sequences": ["Human: ", "Assistant: "],
}
```

### リソースの削除

cli および CDK を利用されている場合、`cdk destroy`を実行してください。そうでない場合は[CloudFormation](https://console.aws.amazon.com/cloudformation/home)へアクセスし、手動で`BedrockChatStack`および`FrontendWafStack`を削除してください。なお`FrontendWafStack`は `us-east-1` リージョンにあります。

### 言語設定について

このアセットは、[i18next-browser-languageDetector](https://github.com/i18next/i18next-browser-languageDetector) を用いて自動で言語を検出します。もし任意の言語へ変更されたい場合はアプリケーション左下のメニューから切り替えてください。なお以下のように Query String で設定することも可能です。

> `https://example.com?lng=ja`

### フロントエンドのローカルでの開発について

現在このサンプルでは、`cdk deploy` された AWS リソース（API Gateway、Cognito など）を使用してローカルでフロントを立ち上げながら改変作業を加えることができます。

1. [Deploy using CDK](#deploy-using-cdk) を参考に AWS 環境上にデプロイを行う
2. `frontend/.env.template` 複製し `frontend/.env.local` という名前で保存する。
3. `.env.local` の中身を `cdk deploy` の出力結果（`BedrockChatStack.AuthUserPoolClientIdXXXXX` など）を見ながら穴埋めしていく
4. 下記コマンドを実行する

```zsh
cd frontend && npm run dev
```

### ストリーミングの利用

現在、環境変数として `VITE_APP_USE_STREAMING` というのをフロントエンド側で指定しています。バックエンドをローカルで動かす場合は `false` に指定してただき、AWS で動かす場合は `true` にすることを推奨します。  
Streaming を有効化すると文章生成結果がストリーミングされるためリアルタイムで文字列が生成されていきます。

### コンテナを利用したローカルでの開発について

[docker-compose.yml](../docker-compose.yml) を利用することで、フロントエンド/バックエンド API/DynamoDB Local をローカル環境で動かし開発を行うことができます。

```bash
# コンテナのビルド
docker compose build

# コンテナの起動
docker compose up

# コンテナの停止
docker compose down
```

### Kendra を利用した RAG について

本サンプルでは Kendra を利用した RAG は実装しておりません。実導入する場合、アクセスコントロールポリシーやデータコネクタの有無、接続先データソースの認証・認可方法は組織により多様なため、シンプルに一般化することが難しいためです。実用するにはレイテンシーの低下やトークン消費量の増加などのデメリットや、検索精度を検証するための PoC が必須であることを考慮する必要があるため、以下のアセットを活用した PoC をおすすめします。

- [generative-ai-use-cases-jp](https://github.com/aws-samples/generative-ai-use-cases-jp) (In Japanese)
- [simple-lex-kendra-jp](https://github.com/aws-samples/simple-lex-kendra-jp) (In Japanese)
- [jp-rag-sample](https://github.com/aws-samples/jp-rag-sample) (In Japanese)
