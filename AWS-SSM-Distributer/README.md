# AWS SSM Distributor を使った Falcon センサーインストール手順

## 概要

AWS Systems Manager (SSM) の Distributor パッケージを使用して、EC2 インスタンスに CrowdStrike Falcon センサーを自動インストールする手順です。

State Manager Association を使用することで、**既存のインスタンス**と**新規作成されたインスタンス**の両方に自動的にセンサーをインストールできます。


本ページは最低限の手順を日本語で説明した簡易ガイドです。詳細は、以下の英語版をご確認ください。
https://github.com/CrowdStrike/aws-ssm-distributor/tree/main


---

## 前提条件

- EC2 インスタンスに SSM Agent がインストール・起動していること
- EC2 の IAM インスタンスプロファイル（ロール）に `AmazonSSMManagedInstanceCore` ポリシーがアタッチされていること
- SSM のエンドポイントに到達できること（VPC エンドポイントまたはインターネット経由）

> **SSM を初めて使う場合：**  
> EC2 用の IAM ロールを作成または既存ロールに `AmazonSSMManagedInstanceCore` ポリシーをアタッチし、EC2 のインスタンスプロファイルに設定してください。

---

## 手順

### Step 1: CrowdStrike API キーの作成

1. Falconコンソールで **サポートおよびリソース** > **APIクライアントおよびキー** を開く
2. **APIクライアントを作成** をクリック
3. 以下のスコープを設定する：

   | スコープ             | 権限   |
   | -------------------- | ------ |
   | Installation Tokens  | READ   |
   | Sensor Download      | READ   |

4. 表示された **CLIENT ID**、**SECRET**、**BASE URL** をコピーして保管する

---

### Step 2: API キーを AWS Parameter Store に保存

以下の3つのパラメータを **SecureString** タイプで作成します。  
**CLIENT_ID**、**SECRET**、**BASE_URL** はStep1で作成したものに置き換えてください。

```bash
aws ssm put-parameter \
    --name "/CrowdStrike/Falcon/ClientId" \
    --type "SecureString" \
    --value "CLIENT_ID"

aws ssm put-parameter \
    --name "/CrowdStrike/Falcon/ClientSecret" \
    --type "SecureString" \
    --value "SECRET"

aws ssm put-parameter \
    --name "/CrowdStrike/Falcon/Cloud" \
    --type "SecureString" \
    --value "BASE_URL"
```

---

### Step 3: SSM Automation 用 IAM ロールの作成

> **注意：** このロールは EC2 インスタンスのロールとは**別のロール**です。  
> SSM Automation ドキュメントが Parameter Store を読み取り、センサーをインストールするために使用します。

CloudFormation を使って作成します：

```bash
curl -s -o ./iam-role.yaml \
  "https://raw.githubusercontent.com/crowdstrike/aws-ssm-distributor/main/official-package/cloudformation/iam-role.yaml" \
&& aws cloudformation create-stack \
  --stack-name crowdstrike-distributor-deploy-role \
  --template-body file://iam-role.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters ParameterKey=SecretStorageMethod,ParameterValue=ParameterStore
```

作成されるロール名: `crowdstrike-distributor-deploy-role`

---

### Step 4: State Manager 関連付け の作成（センサーのデプロイ）

AWS コンソールで操作します。

1. **AWS Systems Manager** > **ノードツール** > **ディストリニューター** > **サードパーティ** を開く
2. **FalconSensor-CrowdStrike** パッケージを選択
3. **スケジュールへのインストール** をクリック → `関連付けへの作成` 画面に遷移
4. 各項目を以下のように設定する：

   | 設定項目 | 値 |
   |---------|-----|
   | ドキュメントのバージョン | `ランタイムのデフォルト` |
   | Execution | `レートの制御` |
   | ターゲット > パラメータの選択 | `InstanceIds` |
   | ターゲット > ターゲットを選択します | インストール対象を選択（例：All Instances） |

5. **入力パラメーター** セクションで以下を入力：

   | パラメータ名 | 値 |
   |------------|-----|
   | AutomationAssumeRole | `crowdstrike-distributor-deploy-role` |
   | Action | `Install` （自動で選択済み）|
   | LinuxInstallerParams | オプション: インストール時の追加パラメータ。 --tags= など |
   | WindowsInstallerParams | オプション: インストール時の追加パラメータ。 GROUPING_TAGS= など |
   | SecretStorageMethod | `ParameterStore` (自動で選択済み）|
   | FalconCloud | `/CrowdStrike/Falcon/Cloud` （自動で入力済み）|
   | FalconClientId | `/CrowdStrike/Falcon/ClientId` （自動で入力済み）|
   | FalconClientSecret | `/CrowdStrike/Falcon/ClientSecret` （自動で入力済み）|

6. **関連付けの作成** をクリック  

以上で、自動でセンサーがインストールされます。Falconコンソールにてセンサーが接続されていることをご確認ください。


---

## アンインストール

既存の **ステートマネージャー** > **関連付け** の中で `Action` パラメータを `Uninstall` に変更して実行することでアンインストールできます。  
アンインストール後、**関連付け**  を削除してください。


---

## 注意事項

- センサーのバージョン管理（アップグレード・ダウングレード）は Falconコンソール の **センサー更新ポリシー** で行います。Distributor パッケージはインストールのみを担当します。
