## Papyrus Invoice

RDSから商品情報を取得し、GUI上の納品リストから納品書をPDF出力するポートフォリオ。

[![ECR Push](https://github.com/Nickelth/papyrus-invoice/actions/workflows/ecr-push.yml/badge.svg)](../../actions)
[![ECS Deploy](https://github.com/Nickelth/papyrus-invoice/actions/workflows/ecs-deploy.yml/badge.svg)](../../actions)
[![ECS Scaling](https://github.com/Nickelth/papyrus-invoice/actions/workflows/ecs-scale.yml/badge.svg)](../../actions)
[![ALB Smoke](https://github.com/Nickelth/papyrus-invoice/actions/workflows/alb-smoke.yml/badge.svg)](../../actions)
[![Audit Evidence](https://github.com/Nickelth/papyrus-invoice/actions/workflows/audit-evidence.yml/badge.svg)](../../actions)
[![RDS Snapshot & Restore Drill](https://github.com/Nickelth/papyrus-invoice/actions/workflows/rds-snapshot.yml/badge.svg)](../../actions)

### ルール

- README は**300行未満**、図鑑は docs。
- 詳細なコマンド羅列は`docs/(date %Y%m%d).md`に置く。
- 実行結果スクショや長表は S3 の成果物か `docs/` にリンクだけ。
- 変更履歴は **CHANGELOG** 系統に一元化。README の「変更履歴」セクションは禁止。

### ディレクトリ構成

```plaintext
papyrus-invoice/
├── Dockerfile
├── docker-compose.yml
├── run.py
├── init.sql
├── requirements.txt
├── .env.dev
├── .env.prd
├── infra/
│   ├── 10-rds/
│   ├── 20-alb/
│   └── 30-monitor/
├── papyrus/
│   ├── blueprints/
│   │   ├── dbcheck.py
│   │   └── healthz.py
│   ├── __init__.py
│   ├── api_routes.py
│   ├── auth_routes.py
│   ├── auth.py
│   ├── db.py
│   ├── routes.py
│   └── config_runtime.py
├── templates/
├── static/
└── docs/
    └── evidence/
```

### 開発環境起動時

```bash
docker compose --env-file .env.dev build --no-cache --progress=plain
```

### 10. RDS起動

Terraformで起動

起動コマンドは以下の「運用フロー 1. plan / apply の実行と証跡化」を参照。

[10-rds/README](infra/10-rds/README.md)

### 20. ALB スモーク

ALB常時起動は非常に高価なため、Terraform構築→疎通確認→即時破壊のCIを用いてALB接続テストを実施する。
ECSタスク起動(`desire=1`)、RDS起動が前提となる。

[20-alb/README](infra/20-alb/README.md)

### 30. CloudWatch Alarm IaC監査

リポジトリクローン後、CloudShell上でコマンド入力

#### CloudWatch Alarm監査体制をIaCで構築

検知条件
- ECS メモリ >80% (平均2/5分)
- ALB 5xx% >1 (Sum 2/5分)
- TargetResponseTime p90 >1.5s

以下コマンドを一度のみ実行

```bash
cd infra/30-monitor
terraform init -input=false
terraform validate
terraform plan   -var-file=dev.tfvars | tee "$EVID/$(date +%Y%m%d_%H%M%S)_monitor_tf_plan.log"
terraform apply  -var-file=dev.tfvars -auto-approve \
  | tee "$EVID/$(date +%Y%m%d_%H%M%S)_monitor_tf_apply.log"
```

Terraform構築成功ログ： [monitor_tf_apply.log](docs/evidence/20251029_021236_monitor_tf_apply.log)

### Github Actions CI/CD

- **Build & Push to ECR** / `ecr-push.yml`: 
  - `master`ブランチデプロイ時及びタグ(`v*.*.*`)付与時に自動実行。ECRイメージを更新。
- **Deploy to ECS Fargate** / `ecs-deploy.yml`: 
  - Actionsで任意実行。ECSタスクをdesire=1にしてサービス起動。
- **ECS Scale Service** / `ecs-scale.yml`: 
  - Actionsで任意実行。ECSタスクをdesire=0にしてサービス停止。
- **Papyrus Smoke** / `alb-smoke.yml`: 
  - Actionsで任意実行。ALB/TG/SGを作成→疎通→破壊。証跡をリポジトリに保存。
- **CloudTrail & Config (last 24h)** / `audit-evidence.yml`: 
  - Actionsで任意実行。直近24hのCloudTrailとConfigを取得。証跡をリポジトリに保存。

### 完成定義

- [x] **ECS→RDS の疎通 OK**（INSERT 0 1 が証跡に残る）

- [x] **最小スキーマ適用済み**（`init.sql` 投入）

- [x] **観測の入口**として CloudWatch Logs に構造化ログが出ている（JSON1行）

- [x] **IaC 薄切り**（RDS/SG/ParameterGroup だけTerraform化。完全Importは後回し）

- [x] **証跡**：`psql` 接続ログ、SG設定SS、アプリログに接続成功 

- [x] **Parameter Group変更の反映証跡**(再起動含む)、トランザクションのログ1件、再試行ロジックの有無

- [x] **CLI履歴の証跡化**: `script`コマンドか`bash -x`ログ、加えてCloudTrail + Configを記事に添える 
  - `script`コマンド → `alb-smoke.yml`に実装
  - CloudTrail + Config → **Papyrus Smoke** / `alb-smoke.yml`に実装

![AWSアーキテクチャ構成図](./invoice-aws-architect.drawio.svg)

### Q&A

**Q1.** Publicサブネットは2a/2b/2cの3AZに対してPrivateサブネットは2a/2bの2AZで不一致なのはなぜ？

**A1.** ALB疎通改善のためにAZ Cを有効化し、publicを3AZにした。privateは2AZ冗長で十分なので意図的に2AZ止まり。

参考：[20251031-ADR](./docs/20251031-ADR.md)