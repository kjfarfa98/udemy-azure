# Azure 基礎知識メモ

---

## クイックリファレンス：AWS ↔ Azure 対応表

| AWS | Azure | 備考 |
|---|---|---|
| Management Console（マネコン） | Azure Portal（ポータル） | |
| アカウント（member） | サブスクリプション | 課金・リソースの管理単位 |
| Organizations | 管理グループ | |
| EC2 | 仮想マシン（VM） | |
| Elastic Beanstalk | App Service | |
| Fargate / ECS | ACI / AKS | |
| Lambda | Azure Functions | |
| S3（サービス全体） | ストレージアカウント内のBlob Storage | |
| S3バケット | コンテナ | |
| S3オブジェクト | Blob | |
| EFS | File Storage | |
| SQS | Queue Storage | |
| DynamoDB | Table Storage | ※簡易版 |
| S3 Glacier | Archive層 | |
| VPC | VNet | |
| セキュリティグループ | NSG（Network Security Group） | |
| VPC Peering | VNet Peering | |
| Direct Connect | ExpressRoute | |
| IAM + IAM Identity Center + Cognito | Entra ID | Azureは1サービスに統合 |
| Reserved Instances（RI） | Azure Reservations | |
| Cost Explorer | Azure Cost Management | AzureはRが無料 |
| CloudFormationスタック | リソースグループ（削除用途） | |
| Cost Allocationタグ | リソースグループ（コスト管理） | |
| プレイスメントグループ（Spread） | 可用性セット | |

---

## Azureの基本構造

### アカウント・サブスクリプションの階層

```
管理グループ（Management Group）  ← AWSのOrganizationsに相当
└── サブスクリプション            ← AWSのアカウント（member）に相当。課金・リソースの管理単位
    └── リソースグループ
        └── リソース
```

| AWS | Azure |
|---|---|
| AWSアカウント（root/管理） | Azureアカウント（Microsoft/Entra IDのログインID） |
| AWSアカウント（member） | サブスクリプション |
| AWS Organizations | 管理グループ |

**AWSとの大きな違い**
- AWSは環境ごとにアカウントを分離する（別メールアドレス・別ログイン）
- Azureは**1つのログインIDのまま、サブスクリプションを切り替えて**環境を分離できる

```
Azureアカウント（1つのログインID）
├── サブスクリプション: myapp-dev
├── サブスクリプション: myapp-staging
└── サブスクリプション: myapp-prod
```

### 環境分離のベストプラクティス

| 分離レベル | 手段 | 備考 |
|---|---|---|
| 本番・非本番の分離 | **サブスクリプションを分ける** | AWSのアカウント分離と同じ考え方 |
| 同一環境内の機能分離 | リソースグループを分ける | PJ単位・ライフサイクル単位など |
| 大規模・複数チーム | 管理グループでサブスクリプションをまとめる | |

- リソースグループだけで本番・非本番を分けるのは**アンチパターン**（学習・検証用途のみ許容）

### AzureとAWSの設計思想の違い

| | Azure | AWS |
|---|---|---|
| 設計思想 | まとめて管理（エンタープライズ寄り） | 独立したサービス（マイクロサービス寄り） |
| 歴史的背景 | Microsoftのオンプレ文化（1サーバーで複数機能）をクラウドに持ち込んだ | Amazonの社内インフラをサービス化。「1つのことをうまくやる」思想 |
| メリット | 認証・課金が一元管理で楽 | サービスごとに細かく権限・スケールを制御できる |
| デメリット | アカウント単位のリスクが大きい | サービスが増えると管理が煩雑 |

この設計思想の違いはストレージだけでなく、Azure全体に通じる特徴。

---

## 管理・構成

### リソースグループ（Resource Group）
- AWSに直接対応するものがない概念（サブネットとは全く別物）
- VMやストレージ・ネットワークなど、あらゆるリソースを束ねる**論理的な入れ物**（ネットワーク的な意味はない）
- AWSではタグやCloudFormationスタックで代替していたものが、Azureでは**第一級の概念**として存在

**できること**

| 用途 | AWSで近いもの |
|---|---|
| まとめて削除 | CloudFormationスタック |
| コスト管理 | Cost Allocationタグ |
| 権限管理（RBAC） | IAMポリシー + タグベースのアクセス制御 |

**特性**
- 異なるリージョンのリソースを混在できる（ただしリソースグループ自体は特定リージョンに属する）
- リソース作成後に別のリソースグループへ移動できる（一部のリソースは移動不可）
- 設定したRBACは配下のリソース全体に継承される
- **学習用は1つのリソースグループにまとめる**（`rg-udemy-study` など）→ 削除するだけで一括クリーンアップ

**実務での使い方**

| パターン | 例 | 用途 |
|---|---|---|
| プロジェクト単位 | `rg-ec-site`, `rg-admin` | コスト管理・PJ終了時の一括削除 |
| ライフサイクル単位 | `rg-myapp-persistent`, `rg-myapp-compute` | 永続データと再作成可能なリソースを分けて保護 |

### コスト管理

**Azure Cost Management** ≒ AWS Cost Explorer

| | AWS Cost Explorer | Azure Cost Management |
|---|---|---|
| 料金 | 有料（クエリごとに課金） | **無料** |
| スコープ | AWSアカウント単位 | サブスクリプション・リソースグループ単位で柔軟に絞れる |
| 予算管理 | AWS Budgets（別サービス） | Cost Management内に統合 |

**Azure Reservations** ≒ AWS Reserved Instances

| | AWS | Azure |
|---|---|---|
| 名称 | Reserved Instances（RI） | Azure Reservations |
| 割引率 | 最大72%程度 | 最大72%程度 |
| 期間 | 1年・3年 | 1年・3年 |

- 課金はすべてサブスクリプション単位に集約される
- 管理グループ単位でReservationsを複数サブスクリプションに共有できる（AWSのOrganizations RI共有と同じ考え方）

---

## 認証・認可

### Entra ID（旧称：Azure Active Directory / Azure AD）
- AWSのIAM・IAM Identity Center・Cognitoを**1つにまとめたようなサービス**

| 用途 | AWSで近いもの |
|---|---|
| 社員・管理者のログイン管理 | IAM Identity Center（旧SSO） |
| AzureリソースへのアクセスのID管理 | IAM |
| 外部ユーザー・アプリのログイン | Cognito |

**AWSのIAMとの大きな違い**
- AWSのIAMは「AWSリソースへのアクセス制御」に特化
- Entra IDは**企業全体のID管理基盤**（Azure・Microsoft 365・SaaSすべてをカバー）
- Azureアカウントを作成した時点でEntra IDテナントが自動作成され、自分のユーザーが登録される

**実務でのイメージ**
```
社員がPCにログイン → Entra IDで認証
→ Azure Portal / Microsoft 365 / 社内SaaSにSSO可
```

### Active Directory（AD）との関係

**ADとは**：オンプレの社内サーバーで動くID管理システム。社員アカウント1つで社内PC・ファイルサーバー・社内システムにアクセスできる仕組みを管理。

| | Active Directory（AD） | Entra ID |
|---|---|---|
| 場所 | オンプレ（社内サーバー） | クラウド（Azure上） |
| 主な用途 | 社内ネットワーク内の認証 | インターネット経由・クラウドの認証 |

**ADだけだと困ること**：ADは社内ネットワーク前提のため、テレワーク・Microsoft 365・SaaSへのアクセスができない。→ Entra IDが必要。

**最も一般的な構成：AD + Entra IDの併用**
```
オンプレ AD（社内サーバー）
        ↕ 同期（Entra Connect）
Entra ID（クラウド）
```
- **Entra Connect** でADとEntra IDを自動同期
- 管理者はADだけ操作すればよい（新入社員追加・退職者無効化もADだけでOK）
- 同じID・パスワードでオンプレもクラウドも使える

| パターン | 内容 |
|---|---|
| ADのみ | 古い企業。クラウド未活用 |
| **AD + Entra ID** | **最も一般的。オンプレとクラウドを橋渡し** |
| Entra IDのみ | クラウドネイティブな企業（スタートアップなど） |

---

## コンピューティング

### 仮想マシン（Virtual Machine / VM）
≒ **AWS EC2**

**AWSとの違い**
- 命名：AWSは `t3.micro`、Azureは `Standard_B1s` のような形式
- **⚠️ 停止時の課金**：AWSはインスタンス停止で課金停止。Azureは「停止」だけでは課金が続く。**「割り当て解除（Deallocate）」** にしないと課金が止まらない
- OSディスク：VM削除時にディスクが残る場合があり、ディスク課金が続くので削除時に要確認
- パブリックIP：AWSのElastic IPに相当する「パブリックIPアドレス」リソースを独立してアタッチする形

学習用おすすめサイズ：`Standard_D2als_v6`（コストが低く汎用タイプ）

### App Service
≒ **AWS Elastic Beanstalk**（コードをデプロイするだけでサーバー管理不要のPaaS）

| App Serviceの使い方 | AWSで近いもの |
|---|---|
| コードをそのままデプロイ | Elastic Beanstalk |
| コンテナをデプロイ | App Runner |
| 関数単位で動かす | Lambda ※Azureでは別途 Azure Functions がある |

- Fargate（ECSのコンテナ実行基盤）に近いのは **ACI / AKS**（App Serviceより抽象度が低い）
- **App Service Plan** がセットで必要 — サーバーのスペック・料金プランに相当
- 複数のアプリを1つのApp Service Planで動かせる
- スロット機能（Blue/Greenデプロイ）が標準で使える

---

## ストレージ

### ストレージアカウント（Storage Account）
- AWSのS3単体ではなく、**複数のストレージサービスをまとめた入れ物**
- 認証・課金・アクセスキーはストレージアカウント単位で一元管理
- ストレージアカウント名はグローバルで一意
- ⚠️ アクセスキーが漏れると全サービスへのアクセスが危険にさらされる

| ストレージアカウント内のサービス | AWSで近いもの |
|---|---|
| **Blob Storage** | S3 |
| **File Storage** | EFS |
| **Queue Storage** | SQS |
| **Table Storage** | DynamoDB（簡易版） |

**構造イメージ**
```
ストレージアカウント（直接対応するAWSリソースなし）
└── コンテナ（≒ S3バケット）
    └── Blob（≒ S3オブジェクト）
```
- URL：`https://<storageaccount>.blob.core.windows.net/<container>/<blob>`
- Blob Storageは独立したリソースではなく、ストレージアカウントを作った時点で使えるエンドポイント
- コンテナ = S3バケット（Dockerのコンテナとは全くの別物）
- ディレクトリは存在せず、Blob名に `/` を含めた仮想パスで階層を表現（S3と同じ）
- **Blob** = Binary Large OBjectの略

**Blobのアクセス層（≒ S3ストレージクラス）**

| Azure | AWSで近いもの | 用途 |
|---|---|---|
| **Hot** | S3 Standard | 頻繁にアクセス |
| **Cool** | S3 Standard-IA | 月1回程度 |
| **Cold** | S3 Standard-IA（より安価） | 数ヶ月に1回程度 |
| **Archive** | S3 Glacier | ほぼアクセスしない長期保存（取り出しに最大15時間） |

```
保存コスト:   Hot > Cool > Cold > Archive
アクセスコスト: Hot < Cool < Cold < Archive
```

- AWSはバケット単位でクラス設定するが、Azureは**Blobファイル単位でアクセス層を変更できる**

---

## ネットワーク

### VNet（Virtual Network）
≒ **AWS VPC**

| | AWS VPC | Azure VNet |
|---|---|---|
| サブネット間通信 | ルートテーブルで制御が必要 | **同一VNet内は自動でルーティングされる** |
| リージョンをまたぐ | 不可 | 不可 |
| 接続 | VPC Peering | VNet Peering |
| オンプレ接続 | Direct Connect / VPN | ExpressRoute / VPN Gateway |

- 細かくサブネット間通信を制御したい場合は **NSG（Network Security Group）** を使う（≒ AWSのセキュリティグループ）

---

## リージョンと可用性

### リージョン（Region）
AWSと同じ考え方。

### Availability Zone（AZ）

| | AWS | Azure |
|---|---|---|
| 指定方法 | `us-east-1a` など | ゾーン番号（1・2・3） |
| ゾーンの物理対応 | アカウントをまたいで一致 | **サブスクリプションをまたぐと不一致**（意図的にずらしている） |

```
Aさんのサブスクリプション   Bさんのサブスクリプション
ゾーン1 → A棟              ゾーン1 → B棟  ← 同じ「1」でも物理的に別の場所
ゾーン2 → B棟              ゾーン2 → C棟
ゾーン3 → C棟              ゾーン3 → A棟
```

- **理由**：全員がゾーン1を選びがちなため、Microsoftが負荷を均等分散させるために意図的にずらしている
- 全リージョンがAZをサポートしているわけではない（例：Japan East → あり、Japan West → なし）

### 可用性セット（Availability Set）
≒ **AWSのプレイスメントグループ（Spread）**

AZが使えないリージョンで、**同一データセンター内**での冗長性を確保する仕組み。

| 概念 | 意味 |
|---|---|
| **障害ドメイン（Fault Domain）** | 電源・NWスイッチを共有する物理ラック単位。異なるラックにVMを分散し障害の影響を局所化 |
| **更新ドメイン（Update Domain）** | Microsoftがメンテナンス（再起動）を行う単位。異なるドメインに分散し同時再起動を防ぐ |

| | 可用性ゾーン | 可用性セット |
|---|---|---|
| 分散範囲 | データセンター間 | 同一データセンター内 |
| 対応リージョン | AZありのみ | すべて |
| 冗長性 | 高い | 中程度 |
| 使いどころ | **基本こちらを優先** | AZが使えない場合のみ |

---

## Azure Portal

≒ **AWS マネコン（Management Console）**

- ブラウザからGUIでリソースを管理するWebインターフェース
- カスタマイズ可能なダッシュボードが標準で充実（AWSより統一感のあるUI）
- 検索バーからリソース名・サービス名で直接飛べる
- 言語設定はヘッダの設定アイコン（歯車）から変更可能

---

## 学習時の注意（無料試用版 Free Trial）

- クレジット（$200/30日）が残っていても**クォータ制限は別管理**なので注意
- クォータは**リージョン単位で独立**して管理される
- 制限はエラーコード（`NotAvailableForSubscription` など）で明示される
- **詰まったらまずリージョンを変えてみる**（例：Japan East → NG → Japan West → OK）
- 人気リージョンほど無料枠が絞られる傾向あり
- 根本解決は**従量課金（Pay-As-You-Go）へのアップグレード**（$200クレジットは引き継がれる）

**リソース削除の習慣**

| リソース | 課金停止の方法 |
|---|---|
| 仮想マシン | **割り当て解除（Deallocate）**（停止だけではNG） |
| App Service | 削除する |
| その他 | 削除する（OSディスクなど残存リソースにも注意） |
