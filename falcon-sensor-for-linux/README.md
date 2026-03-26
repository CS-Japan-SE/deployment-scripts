# Falcon sensor for Linux インストールスクリプト 利用ガイド

以下URLのスクリプトを用いてLinux環境へFalconセンサーをインストールする方法を説明します。  
https://github.com/CrowdStrike/falcon-scripts/tree/main/bash/install  

本ページは日本語の簡易ガイドです。詳細な使用方法は、スクリプトの英語ページをご確認ください。



# 動作概要

スクリプトは以下の処理を行います。

1. CrowdStrike API への OAuth2 認証
2. 対象OSに適合する最新センサーパッケージのダウンロード
3. センサーのインストール
4. CID の取得とセンサーの登録（`falconctl -s`）
5. サービスの起動（`systemctl restart falcon-sensor` / `service falcon-sensor restart`）  

> **重要な動作仕様：**  
> スクリプトは実行環境が **AWS EC2 であることを自動検出** します。  
> EC2上で実行した場合、環境変数 `FALCON_CLIENT_ID` / `FALCON_CLIENT_SECRET` が未設定でも、  
> **AWS SSM Parameter Store** から自動的に `FALCON_CLIENT_ID` および `FALCON_CLIENT_SECRET` の取得を試みます。


# 事前準備 - CrowdStrike APIキーの取得
Falconコンソール（**Support and resources** → **API clients and keys**）にて、以下スコープのAPIキーを作成してください。



| スコープ | 権限 | 必要性 | 説明 |
|---------|------|--------|------|
| **Sensor Download** | Read | ✅ 必須 | Falcon Sensor インストールパッケージのダウンロードに必要 |
| **Installation Tokens** | Read | ⚠️ 条件付き必須 | 環境でインストールトークンが強制されている場合に必要 |
| **Sensor update policies** | Read | 🔷 任意 | `FALCON_SENSOR_UPDATE_POLICY_NAME` 環境変数でセンサーアップデートポリシーを指定する場合に必要 |
| **Sensor update policies** | Write | 🔷 任意（アンインストール時） | アンインストールスクリプトが API からメンテナンストークンを自動取得する場合に必要。`FALCON_MAINTENANCE_TOKEN` 環境変数で直接指定する場合は不要。アンインストール保護が有効なセンサーのアンインストールにメンテナンストークンが必要 |
| **Hosts** | Write | 🔷 任意（アンインストール時） | アンインストールスクリプトで `FALCON_REMOVE_HOST=true` を使用する場合に必要 |

> ⚠️ **`Hosts [Write]` の使用について：**  
> `FALCON_REMOVE_HOST=true` でホストをコンソールから削除するよりも、Falcon Console の **Host Retention Policies** の使用を推奨します。

# インストール手順

### パターン A：環境変数にAPIキーを直接記載（テスト・検証環境向け）

以下のスクリプトを実行します。


```bash
#!/bin/bash
export FALCON_CLIENT_ID="<Your_Client_ID>"
export FALCON_CLIENT_SECRET="<Your_Client_Secret>"

curl -L https://raw.githubusercontent.com/crowdstrike/falcon-scripts/v1.12.0/bash/install/falcon-linux-install.sh | bash
```
このスクリプトは、AWS EC2 - User Data のような起動スクリプトに記述しても動作します。
> ⚠️ ただし、セキュリティの観点から、APIキーの直接記述は推奨しません。  
> 本番環境ではAWS SSM parameter storeのように安全にキーを管理できる仕組みをお使いください。  
> AWS SSM parameter store による方法は後述します。


### パターン B：AWS SSM Parameter Store から認証情報を取得（本番環境推奨）
**Step 1：SSM Parameter Store にAPIキーを登録**

```bash
aws ssm put-parameter \
  --name "FALCON_CLIENT_ID" \
  --value "<Your_Client_ID>" \
  --type SecureString \
  --region ap-northeast-1


aws ssm put-parameter \
  --name "FALCON_CLIENT_SECRET" \
  --value "<Your_Client_Secret>" \
  --type SecureString \
  --region ap-northeast-1
```


**Step 2：EC2 IAMロールに SSM権限を追加**

IAMポリシーに以下の権限を追加します：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParameters"],
      "Resource": [
        "arn:aws:ssm:ap-northeast-1:<Account_ID>:parameter/FALCON_CLIENT_ID",
        "arn:aws:ssm:ap-northeast-1:<Account_ID>:parameter/FALCON_CLIENT_SECRET"
      ]
    }
  ]
}
```

**Step 3：User Data の設定**

User Dataに以下を記載します。
スクリプトが EC2 であることを自動検出し、SSM Parameter Store からAPIキーを自動取得します。

```bash
#!/bin/bash

curl -L https://raw.githubusercontent.com/crowdstrike/falcon-scripts/v1.12.0/bash/install/falcon-linux-install.sh | bash
```



### センサーの動作確認

プロセスの確認
```
ps -e | grep falcon
```

CID, AID, Version, RFM状態の確認
```
/opt/CrowdStrike/falconctl -g --cid --aid --version --rfm-state
```


# アンインストール手順
以下のコマンドを実行します。
```
curl -L https://raw.githubusercontent.com/crowdstrike/falcon-scripts/v1.12.0/bash/install/falcon-linux-uninstall.sh | bash

```


# 主要な環境変数
必要に応じて追加の環境変数を設定してください。本スクリプトはこれら環境変数を自動で読み込みます。

| 環境変数 | デフォルト | 説明 |
|---------|-----------|------|
| `FALCON_CLOUD` | 自動検出 | クラウドリージョン（`us-1` / `us-2` / `eu-1` / `us-gov-1` / `us-gov-2`） |
| `FALCON_CID` | 自動取得 | カスタマーID（省略するとAPIから自動取得） |
| `FALCON_TAGS` | 未設定 | センサーグループタグ（例：`"env:prod,team:infra"`） |
| `FALCON_SENSOR_UPDATE_POLICY_NAME` | 未設定 | センサーアップデートポリシー名 |
| `FALCON_PROVISIONING_TOKEN` | 自動取得 | プロビジョニングトークン（省略するとAPIから自動取得） |
| `FALCON_APD` | 未設定 | プロキシの有効/無効（`true` / `false`） |
| `FALCON_APH` | 未設定 | プロキシホスト（例：`proxy.example.com`） |
| `FALCON_APP` | 未設定 | プロキシポート（例：`8080`） |
| `FALCON_BACKEND` | `auto` | センサーバックエンド（`auto` / `bpf` / `kernel`） |
| `FALCON_INSTALL_ONLY` | `false` | `true` にするとインストールのみ（登録しない） |
| `FALCON_DOWNLOAD_ONLY` | `false` | `true` にするとダウンロードのみ（インストールしない） |
| `PREP_GOLDEN_IMAGE` | `false` | `true` にするとゴールデンイメージ用に準備（AID削除） |
| `ALLOW_LEGACY_CURL` | `false` | curl 7.55.0 未満を許可する場合 `true` に設定 |

