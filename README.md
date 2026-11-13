# aws-emulator-local

[AWS Emulator](https://docs.aws.amazon.com/emulator/latest/userguide/what-is-service.html) は AWS の API を手元で再現するプログラムです。

## 起動と停止

初回は `.env.example` をコピーして `.env` を作ります。

```bash
cp .env.example .env
```

```bash
docker compose up -d
docker compose down
```

- コンテナを停止するとデータはすべて消えます。
- Intel のマシンでは `.env` の `EMULATOR_ARCH` を `x86_64` にします。
- ホストで `PORT` が使われているとつながりません。`.env` の `PORT` を変えます。

| 項目 | 値 |
| --- | --- |
| `PORT` | `9298` |
| `EMULATOR_TELEMETRY` | `off` |
| `EMULATOR_ARCH` | `aarch64` |

## 接続先

| つなぐ元 | URL |
| --- | --- |
| ホスト | `https://localhost:9298` |
| `aws-emulator` ネットワークのコンテナ | `http://aws-emulator` |

## AWS CLI

```ini
# ~/.aws/config
[profile emulator]
region = ap-northeast-1
endpoint_url = https://localhost:9298

# ~/.aws/credentials
[emulator]
aws_access_key_id = dummy
aws_secret_access_key = dummy
```

```bash
aws --profile emulator --no-verify-ssl sts get-caller-identity
```

## アプリのコンテナ

```yaml
services:
  app:
    environment:
      AWS_ENDPOINT_URL: http://aws-emulator
      AWS_ACCESS_KEY_ID: dummy
      AWS_SECRET_ACCESS_KEY: dummy
      AWS_REGION: ap-northeast-1
    networks:
      - default
      - aws-emulator

networks:
  aws-emulator:
    external: true
```

- エミュレーターを先に起動します。
- S3 はパススタイルにします（JavaScript v3 は `forcePathStyle: true`）。
- 起動のたびにバケットやテーブルを作ります。DynamoDB は `aws dynamodb wait table-exists` で作成を待ってから書き込みます。

## Terraform

backend は local にします。本物の AWS の state を使う構成には向けません。

```hcl
terraform {
  backend "local" {}
}

provider "aws" {
  region                      = "ap-northeast-1"
  access_key                  = "dummy"
  secret_key                  = "dummy"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true
  insecure                    = true

  endpoints {
    s3       = "https://localhost:9298"
    dynamodb = "https://localhost:9298"
    sts      = "https://localhost:9298"
  }
}
```

## エミュレーターで確かめられないこと

- IAM ポリシーとリソースベースのポリシー（[Security](https://docs.aws.amazon.com/emulator/latest/userguide/security.html)）
- 複数のアカウントやリージョンにまたがる構成
- 再現の度合いはサービスごとに違います（[Releases](https://github.com/aws/aws-emulator/releases) の `scores.json.gz`）

## 守ること

- 本物の認証情報とデータを入れません。認証を切っているので、ネットワーク内のコンテナは誰でも操作できます。
- ポートはホストの `127.0.0.1` にだけ開けます。
