# Nova Sonic Contact Center with Vonage

著者: Andrew Young, Reilly Manton

このソリューションは、カスタマーサポートの文脈において Amazon Bedrock の Nova 音声対音声モデルのインタラクションを監視・強化するための包括的な分析ダッシュボードを紹介します。ダッシュボードは、リアルタイム感情分析と視覚的トレンドグラフ、全体通話感情のドーナツチャート可視化、AI を活用したエージェントガイド、およびエージェント/顧客の発話時間割合や応答時間などの主要業績評価指標を提供します。すべてのフロントエンド分析は Amazon Nova Lite モデルで動作し、バックエンド S2S モデルはナレッジベース統合を備えており、通信事業者のカスタマーサポート業務を強化する強力なツールキットとなっています。

## セットアップ

### 前提条件
- npm
- Nova S2S モデルへのアクセス権を持つ AWS アカウント
- AWS CLI 認証情報
- マイクとスピーカー
- Vonage アカウントと電話番号

ルートフォルダから `npm install` を実行します

### 環境変数ファイル (.env) の準備

サーバーは `src/server/.env` から環境変数を読み込みます。  
`src/server/.env.example` をコピーして `.env` を作成し、実際の値を記入してください。

```bash
cp src/server/.env.example src/server/.env
# その後、src/server/.env を編集して AWS 認証情報などを設定
```

**注意:** `.env` には AWS シークレットキーなどの機密情報が含まれるため、Git にコミットしないでください（`.gitignore` に追加することを推奨）。

### クライアント（フロントエンド）のセットアップ
1. `src/server/deploy/` 内で `cdk deploy` を実行して EC2 サーバーをデプロイします
3. クライアントアプリを起動するには `npm run client` を実行します

### サーバー（バックエンド）のセットアップ
このサーバーの実行方法は 2 つあります。Vonage / Twilio / SIP エンドポイントなどの電話プロバイダ経由、またはウェブ経由です。

インバウンドテレフォニーが必要な場合は、EC2 インスタンス上でサーバーをリモート実行し、SSL を使用することを強く推奨します。`ngrok` や `localtunnel` などの他の方法もありますが、EC2 が最も安全であり、当社は他のアプローチを推奨しません。これらのアプローチはウェブと SIP トラフィックを受け付けます。

インバウンドテレフォニーが不要な場合、またはサーバーをテストしたいだけの場合は、単に `localhost` でサーバーをホストできます。これはウェブトラフィックのみを受け付けます。

#### オプション 1: インターネット経由でサーバーをローカル実行する
1. 環境変数を設定します。`.env` ファイル（`src/server/.env`）に記入するか、シェルで `export` してください。

   `.env` を使用する場合（推奨）:
   ```bash
   cp src/server/.env.example src/server/.env
   # src/server/.env を編集して AWS_ACCESS_KEY_ID などを設定
   ```

   またはシェルで直接 export する場合:
   ```bash
   > export AWS_ACCESS_KEY_ID="your-access-key"
   > export AWS_SECRET_ACCESS_KEY="your-secret-key"
   > export AWS_DEFAULT_REGION="us-east-1"
   ```

2. `SERVER_URL` を `.env` に設定します（ウェブサーバーの場所、例: `localhost` や `mydomain.example.com` などの DNS URL）。
3. ウェブアプリケーションから localhost 経由でサーバーにアクセスしたい場合は、これ以上の作業は不要です。クライアントを使用してアプリにアクセスしてください。インターネット経由でのインバウンド通話を受け付けることはできません。インバウンド通話を受け付ける必要がある場合は、代わりに SSL を使用してリモートでサーバーを実行してください（下記参照）。

#### オプション 2: SSL を使用してリモートでサーバーを実行する（インバウンドテレフォニーあり）
1. `src/server/` の全内容を `scp` で転送します
```bash
> scp -r -i <path-to-PEM-file> ./*  ec2-user@<ec2-ip-address>:~
```
2. アプリのデプロイ時に作成した秘密鍵を使用して EC2 インスタンスに SSH でログインします。
```bash
> ssh -i <path-to-PEM-file> ec2-user@<ec2-ip-address>
```
3. SSH サーバー上で `npm install` を実行してすべてのパッケージをインストールします。
4. ネットワーク設定がポート `443` とポート `3000` へのトラフィックを許可していることを確認してください（CDK パッケージではデフォルトで許可されています）。
5. 環境変数を設定します。EC2 上で `.env` ファイル（`src/server/.env`）に記入するか、シェルで `export` してください。

   `.env` を使用する場合:
   ```bash
   cp src/server/.env.example src/server/.env
   # src/server/.env を編集して AWS 認証情報などを設定
   ```

   またはシェルで直接 export する場合:
   ```bash
   > export AWS_ACCESS_KEY_ID="your-access-key"
   > export AWS_SECRET_ACCESS_KEY="your-secret-key"
   > export AWS_DEFAULT_REGION="us-east-1"
   ```

6. アプリケーションを実行します。
```bash
> sudo AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN ts-node server.ts
```

##### Vonage のセットアップ
1. vonage.com にアクセスします
2. アカウントをお持ちでない場合は、Vonage アカウントにサインアップし、十分な残高があることを確認してください。
3. アカウントをお持ちの場合は、Vonage Communications API ダッシュボードにログインします。
4. 左側のメニューから Applications をクリックします。
5. メイン画面で `+ Create new application` をクリックします
6. Capabilities で Voice 機能を有効にします。
7. Answer URL には `<your-hostname>/vonage/answer` への `HTTP GET` を設定します
7. Event URL には `<your-hostname>/vonage/event` への `HTTP POST` を設定します
8. Fallback URL には `<your-hostname>/vonage/fallback` への `HTTP POST` を設定します
9. アプリケーションに電話番号を作成して関連付けます。電話番号の購入が必要な場合があります。

### アプリの実行
1. サーバーが環境変数を設定した状態で実行され、トラフィックを受信できることを確認してください。
2. アプリで作成した電話番号に電話をかけます。
3. その他の参加者を通話に招待したい場合は、サーバーログに表示される call ID をクライアント（フロントエンド）ダッシュボードに貼り付けてください。
4. これで通話が機能し、デバイスで話すとクライアントとサーバーの両方がそれを取得します。

### カスタマイズ
- AI のインサイトとユーザー感情プロンプトをカスタマイズするには、`ui-stream/src/bedrockSentimentAnalyzer.ts` を参照してください