# S3 Deploy Custom Resource

S3からzipファイルをダウンロードして静的ウェブサイトをデプロイするCloudFormationカスタムリソース。

## 概要

このテンプレートは以下の機能を提供します：

- S3バケットからzipファイルを取得
- zipファイルを展開してウェブサイト用S3バケットにデプロイ
- 静的ウェブサイトホスティングの設定
- パブリックアクセス許可
- スタック削除時の自動クリーンアップ

## アーキテクチャ

```
ソースS3バケット → Lambda関数 → ウェブサイトS3バケット → 静的ウェブサイト
     ↓              ↓              ↓
  zipファイル    展開・アップロード   HTMLファイル等
```

## 構成要素

- **Lambda関数**: zipファイルの展開とS3へのアップロード処理
- **IAMロール**: Lambda関数のS3アクセス権限
- **S3バケット**: 静的ウェブサイトホスティング用
- **バケットポリシー**: パブリック読み取りアクセス許可
- **カスタムリソース**: デプロイメントのトリガー

## パラメータ

| パラメータ | 説明 | デフォルト値 |
|-----------|------|-------------|
| SourceBucket | zipファイルが格納されているS3バケット名 | bucketname |
| SourceKey | zipファイルのS3キー | sample-website.zip |
| WebsiteBucketName | ウェブサイト用S3バケット名 | (必須入力) |

## 前提条件

1. ソースS3バケットにzipファイルが存在すること
2. zipファイルにindex.htmlが含まれていること
3. 適切なAWS認証情報が設定されていること

## デプロイ方法

### AWS CLIを使用

```bash
# パラメータファイルを作成
cat > parameters.json << EOF
[
  {
    "ParameterKey": "SourceBucket",
    "ParameterValue": "your-source-bucket"
  },
  {
    "ParameterKey": "SourceKey", 
    "ParameterValue": "your-website.zip"
  },
  {
    "ParameterKey": "WebsiteBucketName",
    "ParameterValue": "your-website-bucket"
  }
]
EOF

# スタックをデプロイ
aws cloudformation create-stack \
  --stack-name my-website-deploy \
  --template-body file://static-website-deployment.yaml \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_IAM \
  --region us-west-1
```

### AWS コンソールを使用

1. CloudFormationコンソールを開く
2. 「スタックの作成」を選択
3. テンプレートファイルをアップロード
4. パラメータを入力
5. IAM機能の承認にチェック
6. スタックを作成

## 検証方法

### 1. デプロイ状況の確認

```bash
# スタック状態を確認
aws cloudformation describe-stacks \
  --stack-name my-website-deploy \
  --query 'Stacks[0].StackStatus'

# 出力値を確認
aws cloudformation describe-stacks \
  --stack-name my-website-deploy \
  --query 'Stacks[0].Outputs'
```

### 2. ウェブサイトアクセステスト

```bash
# ウェブサイトURLを取得
WEBSITE_URL=$(aws cloudformation describe-stacks \
  --stack-name my-website-deploy \
  --query 'Stacks[0].Outputs[?OutputKey==`WebsiteURL`].OutputValue' \
  --output text)

# アクセステスト
curl -I $WEBSITE_URL
```

### 3. ファイル確認

```bash
# デプロイされたファイルを確認
BUCKET_NAME=$(aws cloudformation describe-stacks \
  --stack-name my-website-deploy \
  --query 'Stacks[0].Outputs[?OutputKey==`BucketName`].OutputValue' \
  --output text)

aws s3 ls s3://$BUCKET_NAME/
```

### 4. Lambda関数ログ確認

```bash
# Lambda関数名を取得
FUNCTION_NAME=$(aws cloudformation describe-stack-resources \
  --stack-name my-website-deploy \
  --logical-resource-id DeploymentFunction \
  --query 'StackResources[0].PhysicalResourceId' \
  --output text)

# ログを確認
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/lambda/$FUNCTION_NAME"
```

## トラブルシューティング

### よくある問題

1. **ソースバケットアクセスエラー**
   - バケット名とキーが正しいか確認
   - Lambda実行ロールの権限を確認

2. **zipファイル展開エラー**
   - zipファイルの形式が正しいか確認
   - ファイルサイズがLambdaの制限内か確認

3. **ウェブサイトアクセスエラー**
   - バケットポリシーが正しく設定されているか確認
   - index.htmlが存在するか確認

### ログ確認方法

```bash
# CloudWatch Logsでエラーを確認
aws logs filter-log-events \
  --log-group-name "/aws/lambda/$FUNCTION_NAME" \
  --filter-pattern "ERROR"
```

## クリーンアップ

```bash
# スタックを削除（ウェブサイトファイルも自動削除される）
aws cloudformation delete-stack \
  --stack-name my-website-deploy
```

## 注意事項

- Lambda関数のタイムアウトは300秒（5分）
- 大きなzipファイルの場合は処理時間を考慮
- ウェブサイトバケットは完全にパブリックアクセス可能
- スタック削除時にウェブサイトファイルは自動削除される