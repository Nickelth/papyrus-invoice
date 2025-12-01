## 2025-12-01

### 目的

* RDS 手動スナップショット作成～リストア～削除までを GitHub Actions から自動実行できるようにする
* 実行ごとに RTO・接続情報・適用パラメータなどの証跡を `docs/evidence` に自動保存する仕組みを整える
* 証跡用ブランチ＋PR作成までを CI で完結させ、DR 演習の再現性とレビュー性を高める

### 変更内容

* GitHub Actions 用 IAM ロール（`ECRPowerUser`）に RDS 操作用権限を追加

  * 追加した主なアクション

    * `rds:CreateDBSnapshot` / `rds:DescribeDBSnapshots`
    * `rds:RestoreDBInstanceFromDBSnapshot`
    * `rds:DescribeDBInstances` / `rds:DeleteDBInstance`
    * `rds:AddTagsToResource` / `rds:ListTagsForResource`
  * 本番相当インスタンスは引き続き誤削除されないよう、Delete の範囲を一時インスタンス（`papyrus-pg16-dev-restore-*`）に限定するポリシーに整理

* RDS スナップショット＆リストア演習用ワークフローを整備

  * 対象インスタンス: `papyrus-pg16-dev`（`DB_INSTANCE_ID`）
  * 実装した処理フロー

    * `aws rds create-db-snapshot` による手動スナップショット作成
    * `restore-db-instance-from-db-snapshot` による一時インスタンス `papyrus-pg16-dev-restore-<UTCタイムスタンプ>` の作成

      * 不正なオプションだった `--kms-key-id` を削除し、スナップショット側の暗号化設定を継承する形に修正
    * `wait db-instance-available` による復旧完了待ちと経過秒数（RTO）の計測
    * エンドポイント・ポート番号・CA 証明書 ID・適用パラメータグループの取得
    * 一時インスタンス削除（`--skip-final-snapshot`）
  * 証跡 JSON を `docs/evidence` に出力するよう統一

    * ファイル名は `YYYYMMDD_HHMMSS_種別.json` 形式に統一

* 証跡 PR 自動作成の仕組みを追加

  * `dev` ブランチから `evidence/rds-drill-YYYYMMDD-HHMMSS` ブランチを自動作成
  * 証跡 JSON をコミットして push
  * `gh pr create` により `dev` 向け PR を自動作成

    * ラベル `evidence` / `automation` を付与
  * 上記のために以下を実施

    * リポジトリにラベル `evidence`, `automation` を事前作成
    * `Settings > Actions > Workflow permissions` を

      * `Read and write permissions`
      * 「Actions による PR 作成を許可」
        に変更し、`GITHUB_TOKEN` で PR 作成が可能なように調整

### 証跡一覧

今回の疎通確認実行（2025-12-01）で生成された代表的な証跡:

* `docs/evidence/20251201_075653_rds_snapshot_start.json`

  * 手動スナップショット作成リクエストのレスポンス
* `docs/evidence/20251201_075929_rds_snapshot_facts.json`

  * 作成されたスナップショットの詳細情報
* `docs/evidence/20251201_075934_rds_restore_start.json`

  * 一時インスタンス復元リクエストのレスポンス
* `docs/evidence/20251201_080815_rds_pg_applied.json`

  * 一時インスタンスに適用されたパラメータグループ情報
* `docs/evidence/20251201_080815_rds_restore_facts.json`

  * 一時インスタンスのエンドポイント・ポート・CA 証明書 ID・RTO 秒数
* `docs/evidence/20251201_080816_rds_restore_delete.json`

  * 一時インスタンス削除リクエストのレスポンス
* GitHub PR

  * タイトル: `evidence: RDS snapshot & restore drill 20251201-075140`
  * ブランチ: `evidence/rds-drill-20251201-075140`
  * ラベル: `evidence`, `automation`

### ロールアウト / 影響

* ロールアウト方法

  * 手動トリガー専用ワークフロー（`workflow_dispatch`）として運用
  * DR 演習を実施したいタイミングで、GitHub Actions 画面から対象ワークフローを実行する
  * 実行後、自動作成された evidence PR をレビューし、内容を確認のうえ必要に応じて `dev` にマージする

* 影響範囲

  * 対象 RDS: `papyrus-pg16-dev` と、そのスナップショットから作成される一時インスタンスのみ
  * 一時インスタンスは毎回 `--skip-final-snapshot` で削除されるため、長期リソース増加は発生しない
  * IAM ロールに RDS 操作用の権限が追加されているが、Delete は本体インスタンスではなく DR 用一時インスタンスに限定しており、プロダクションデータ削除リスクは増加していない

* 想定される運用上の注意点

  * ワークフロー実行中は一時的に RDS インスタンスが増えるため、Account や Region のサービスクォータ（DB インスタンス数）に余裕があることを前提とする
  * IAM ポリシー変更やインスタンス名変更を行った場合は、本ワークフローとポリシーの整合性確認が必要

---

## 2025-11-04

### 目的
- スモーク検証（ALB→ECS→RDS）の**証跡を最小構成で自動収集**し、`dev` へ PR 化。
- 監査用途（CloudTrail/Config）の**24hエクスポートを別ワークフローに分離**して、ノイズ/権限起因の失敗を排除。
- CIの一時資材や冗長ダンプを削減し、**読みやすいエビデンスだけ**を残す。

### 変更内容
- **ワークフロー**
  - 新規：`Audit Evidence`（手動トリガ）。CloudTrail/Config の24hスナップショットを収集し `dev` 宛 PR。
  - 改修：`Papyrus Smoke`
    - `docs/evidence` への**統一出力**（`EVID_DIR`）。  
    - **CloudWatch Logs の1行JSON**を `/healthz` `/dbcheck` について自動収集（`if: always()`）。
    - **AZミスマッチ検知**（ALB有効AZとタスクAZの不一致をFail化）。
    - **カスタム wait healthy**（`describe-target-health` で120秒待機）。
    - **SG重複回避**（`InvalidPermission.Duplicate` を握って続行）。
    - **Listener冪等更新**（DuplicateListenerに強い張り替え処理）。
    - **tfstateのアップロードを廃止**、デバッグ用アーティファクト/ダンプ類を削減。
    - 証跡PR：`evidence/smoke-<timestamp>` ブランチ → **dev宛PR自動作成**。`*.log`/`*.json` をコミット。
    - 細部修正：`dev.auto.tfvars` を plan/apply の参照名と一致させる、`$EVID` を廃止し `EVID_DIR` に統一。
- **IAM（必要権限の追加）**
  - **CloudWatch Logs 読み取り**：`logs:FilterLogEvents` ほか。
  - **CloudTrail 読み取り**：`cloudtrail:LookupEvents` ほか。
  - いずれも GitHub Actions 実行ロールに付与（最小権限 or 管理ポリシー）。

### 証跡一覧
- `docs/evidence/20251105_012716_healthz.log`（HTTP 200 / `{"ok":true}`）
- `docs/evidence/20251105_012716_dbcheck.log`（HTTP 200 / `{"ok":true,"inserted":true}` 等）
- `docs/evidence/20251105_012716_cloudwatch_healthz.json` / `20251105_012716_cloudwatch_dbcheck.json`
- `docs/evidence/20251105_012716_healthz_json_line.log` / `20251105_012716_dbcheck_json_line.log`
- （Audit Evidence）`docs/evidence/20251105_012716_cloudtrail_24h_filtered.json`  
  `docs/evidence/20251105_012716_config_recorders.json` / `_config_delivery_channels.json`

### ロールアウト / 影響
- `Papyrus Smoke`：手動実行→`docs/evidence` に証跡出力→**devへPR**（レビュー後マージ）。
- `Audit Evidence`：必要時のみ手動実行し、監査用JSONをPR化。
- 既存本番リソースへの影響：なし（スモーク用ALB/TGは毎回作成・破棄）。
- ロールバック：不要（PRをRevertすれば証跡のみ戻る）。

## 2025-11-03

### 目的
- **観測の入口**を固定化し、/healthz と /dbcheck の到達性を数値で示す。
- **/dbcheck の安定化**（SSL・コネクション管理）により 500/落ちを解消。
- **CIエビデンスをリポジトリ直下に保存**して、CloudShell/面接資料から即参照可能にする。

### 変更内容
- **アプリ (/dbcheck)**
  - 接続プールを `sslmode=require` かつ TCP keepalive 有効で初期化。
  - `OperationalError` 検知時は壊れたコネクションをプールから除外し **1回だけ再接続**してリトライ。
  - レスポンスを単一JSONに統一：`{"ok": true, "inserted": <bool>}`。
  - 失敗時は `{"ok": false, "error": "..."}` を返却し、構造化ログ（JSON 1行）へ出力。
- **CI（alb-smoke.yml）**
  - **Listener→TG の冪等張り替え**（DuplicateListener耐性）。
  - **AZミスマッチ検知**（ALBの有効AZとタスクAZが不一致なら Fail）。
  - **カスタム healthy 待機**（`describe-target-health` ループ）。
  - **SG重複回避**（`InvalidPermission.Duplicate` を握って続行）。
  - **エビデンス保存先を `docs/evidence/` に変更**し、ログをリポ直配置。
  - **dev に対する自動PR**（`evidence/smoke-<timestamp>` ブランチにコミット→PR作成）。
- **ALB/TG**
  - 健診パスを `/healthz`（200–399）で固定。
  - スモーク用ALBのサブネットを **us-west-2c を含む**ように拡張（`PUBLIC_SUBNET_IDS_JSON` 更新）。

### 証跡一覧
- `docs/evidence/20251104_050806_healthz.log`（`HTTP/1.1 200 OK` / `{"ok":true}`）
- `docs/evidence/20251104_050806_dbcheck.log`（`HTTP/1.1 200 OK` / `{"inserted":true}`）
- `docs/evidence/20251105_010050_cloudwatch_healthz.json`（CloudWatch 抜粋）
- `docs/evidence/20251105_010050_healthz_json_line.log`（1行JSONサンプル）
- `docs/evidence/20251105_010050_cloudtrail_24h.json`（直近24hのCloudTrail）
- `docs/evidence/20251105_010050_config_24h.json` (直近24hのConfig)
- （任意）`infra/20-alb/terraform.tfstate`（一時ALB/TGの構成証跡）
- CI Run: `Papyrus Smoke`（Artifacts: `smoke-<run_id>`）

### ロールアウト
- **手順**
  1. CIが作成した PR（`evidence/smoke-<timestamp> -> dev`）をレビューしてマージ。
  2. アプリ修正を含む場合は `ECR push → ECS force-new-deployment` を実行。
  3. `Papyrus Smoke` を手動実行し、`/healthz` `/dbcheck` と TG `healthy` を確認。
- **影響/停止**
  - スモーク用ALB/TGは一時リソース。アプリの再デプロイ時以外は無停止。
- **ロールバック**
  - 直前のタスク定義リビジョンへ戻す（`update-service --force-new-deployment`）。
  - CI側は PR を Revert すれば `docs/evidence` の差分は元に戻る。


## 2025-10-29

### 目的

Papyrus インフラの検証フローを、「説明ではなく証跡で示す」運用へ引き上げる。具体的には次を実施した。

- アプリ健全性と RDS への書き込み可否を、ALB 経由で自動検証できる状態にする
- 検証手順を CI / Terraform に固定し、再現性・監査性を担保する
- PostgreSQL 接続は TLS 必須（PGSSLMODE=require）を構成として恒久化する
- 監視を IaC 管理下に移し、稼働後の可観測性を確保する
- 一時 ALB を CI で「作る→疎通→壊す」まで完全自動化し、コストリークを防止する

これにより、稼働可否・DB 書き込み可否・TLS 強制・監視・リソース破棄を、毎回の自動テストと証跡ログで説明できる。

### 変更内容

#### 1. `/healthz` の実装と疎通確認の自動化

- `papyrus/blueprints/healthz.py` を追加し、DB 非依存で `{"ok": true}` を返す軽量ヘルスを実装。
- Flask 初期化時に `healthz_bp` を登録。ALB/Target Group のヘルスチェックパスを `/healthz` に統一可能な前提を整備。

#### 2. `/dbcheck` の ALB 経由動作検証

- `papyrus/blueprints/dbcheck.py` を Blueprint として正式登録。URL マップに確実に載る形へ整理。
- Fargate 上で `/dbcheck` が HTTP 200 を返し、RDS へ `INSERT ... ON CONFLICT DO NOTHING` により `SKU-APP` を投入できることを確認。
- CI の smoke で ALB 経由アクセスを `curl` し、`HTTP/1.1 200 OK ... {"inserted":true}` を取得・保存。

#### 3. ALB smoke パイプラインの確立（CI 一時本番）

- GitHub Actions `alb-smoke` を整備。Terraform で一時的に ALB / Target Group / Listener / ALB 用 SG を作成し、ECS タスクのプライベート IP:5000 をターゲット登録。
- `aws elbv2 wait target-in-service` によるヘルス化待機後、ALB の DNS に対して `/healthz` と `/dbcheck` を実行し、HTTP ステータスと JSON 本文を証跡化。
- 実行後は `terraform destroy` で ALB/TG/Listener/SG を必ず削除。ALB 名や TG 名はサフィックス付与運用に寄せ、過去残骸との競合を低減。
- セキュリティグループの一時 Ingress（ALB → ECS タスク:5000/tcp）は自動付与・撤収の流れを組み込み中（詳細は「残課題」）。

#### 4. `PGSSLMODE=require` の恒久適用

- ECS タスク定義の再生成ロジックに `{"name":"PGSSLMODE","value":"require"}` を自動付与。
  反映は `register-task-definition` → `update-service --force-new-deployment --enable-execute-command`。
- 反映後、`ecs execute-command` で環境変数を確認し、TLS 必須が常設されている証跡を取得。

#### 5. DB 書き込み経路の二重化検証

- ECS Exec + `psycopg2` の CLI 経路で `SKU-CLI` を RDS へ INSERT。
- アプリ経由 `/dbcheck` で `SKU-APP` を RDS へ INSERT。
- 両データの存在を以下のクエリで確認。

  ```sql
  SELECT sku,name,unit_price
  FROM papyrus_schema.products
  WHERE sku IN ('SKU-CLI','SKU-APP');
  ```

  実測: `ROWS: [('SKU-CLI','health',0), ('SKU-APP','health',0)]`

#### 6. プリフライト（CI 前段チェック）導入

- `from papyrus import create_app` → ルーティング一覧ダンプで `/healthz` `/dbcheck` の欠落を検出した場合は即 Fail。
- RDS 接続メタ情報のドリフト検知を追加。Secrets Manager (`papyrus/prd/db`) の `host`/`port` と、`describe-db-instances` の実体を比較し不一致なら Fail。
- Secrets/SSM 依存で import が失敗するケースは、ランナー側に最低限のダミー環境変数を噛ませる運用（暫定）で回避。

#### 7. 監視 IaC（`infra/30-monitor`）の導入

- `infra/30-monitor/` 配下で CloudWatch Alarm を Terraform 管理化。
  監視メトリクス:

  - ECS メモリ使用率 > 80%（平均、2/5 分）
  - ALB の 5xx レスポンス > 1（合計、2/5 分）
  - TargetResponseTime p90 > 1.5s（2/5 分）
- Alarm は destroy 対象ではなく常設。SNS 通知先を関連付け、`terraform apply` 済み。

#### 8. ドキュメント整備方針（README）

- `infra/10-rds/README.md`: 必須変数、ParamGroup（`rds.force_ssl=1`）と再起動タイムスタンプ管理、DRY RUN→本適用、証跡ファイル一覧を明文化。
- `infra/20-alb/README.md`: 「一時 ALB を作成→疎通→必ず destroy。ALB 放置はコスト燃焼」を運用ルールとして明記。CI アーティファクトの保存方針も明記。
- `infra/30-monitor/README.md`: 監視閾値と SNS 設計意図、IaC 管理方針を記述。

### 証跡一覧

- `*_healthz.log`

  - ALB 経由で `HTTP/1.1 200 OK ... {"ok":true}` を取得。
- `*_dbcheck.log`

  - ALB 経由で `HTTP/1.1 200 OK ... {"inserted":true}` を取得。アプリ経由の RDS への INSERT 成功を証明。
- `*_papyrus_sku_check.log`

  - `ROWS: [('SKU-CLI','health',0), ('SKU-APP','health',0)]` を確認。
- `*_pgsslmode_env.log`

  - ECS Exec による環境変数ダンプで `PGSSLMODE=require` を恒久適用済みであることを確認。
- `infra/30-monitor` の `terraform apply` ログ

  - `Apply complete! Resources: 4 added, 0 changed, 0 destroyed.` を保持。
- `alb-smoke` ワークフローの実行ログ

  - プリフライト `ROUTES: [...]`、Secrets vs RDS ドリフトチェック結果、`register-targets`→`wait target-in-service` 成功、`terraform destroy` の完了記録。

証跡は `docs/evidence/` に保存し、同時に CI アーティファクトとして収集済み。

### ロールアウト

標準リリースパスは次の通り。

1. 新コンテナイメージを ECR に push
2. jq でタスク定義を再生成し、`PGSSLMODE=require` を強制付与
3. `register-task-definition` → `update-service --force-new-deployment --enable-execute-command`
4. RDS 稼働・ECS desiredCount=1 を満たした状態で `alb-smoke` を起動（`workflow_dispatch` で手動実行可）
5. Terraform で一時 ALB/TG/Listener/ALB-SG を作成し、ECS タスクをターゲット登録
6. ALB 経由で `/healthz` と `/dbcheck` を実行し、`{"ok":true}` / `{"inserted":true}` を証跡化
7. `terraform destroy` で ALB/TG/Listener/SG を完全撤収
8. 監視は `infra/30-monitor` を `apply` 済みで常時有効

このフローを通過したイメージは、正常応答・TLS 必須での RDS 書き込み可・監視下・コストリークなし、を満たす。

### 残課題

- [ ] CloudWatchにJSON 1行ログが出ている証跡ファイルがdocs/evidence/にある

- [ ] *_cli.castとCloudTrail/Configの24hエクスポートがdocs/evidence/にある

- [ ] infra/10-rds 20-alb 30-monitorのREADMEが更新済みで、コマンドと証跡の置き場が明記されている

- [ ] alb-smokeでSG二重回避と自前waitが効いており、一発成功のログがある

- [ ] ルートREADMEに“観測”“CLI証跡”の短い説明が追記され、総行数300未満

---

## 2025-10-28

### 目的

Papyrus の「本番相当の最小構成」をCI上で一時的に起動し、ALB経由でFargateタスクへ到達できること、アプリの `/healthz` と `/dbcheck` が正常応答することを証跡付きで確認し、最後にすべて削除することを自動化した。

ALB / Target Group / SG / Fargateタスク / RDS を一時的に立てて HTTP 200 と DB書き込みを確認し、証跡を残してから `terraform destroy` まで行うCIを完成させた。

### 主要変更

1. **`alb-smoke` ワークフロー拡張**

   - `workflow_dispatch` で手動起動可能な `alb-smoke` を強化。
   - Terraformで ALB / Target Group / ALB用SG / タスクSGへの一時Ingress を `apply`。
   - `apply` 後の output を後続ステップで使えるようにエクスポート。

2. **Fargateタスクの動的検出とターゲット登録**

   - CIロールに `ecs:ListTasks` などを追加し、`papyrus-task-service` の RUNNING タスクを特定。
   - ENI からタスクのプライベートIPを取得し、`aws elbv2 register-targets` で Target Group に登録。
   - `aws elbv2 wait target-in-service` で ALB 側のヘルスチェック(InService)まで自動待機。
   - SGは `ecs_tasks_sg_id` を tfvars から渡し、Terraformが「ALB SG -> タスク SG:5000/tcp」の一時Ingressを自動で開閉するよう統一。

3. **疎通テストと証跡取得の自動化**

   - ALB の DNS 名に対して `curl -si http://ALB_DNS/healthz` と `/dbcheck` を実行。
   - ステータスライン / レスポンスヘッダ / ボディを `evidence/` 以下に時刻付きログとして保存。
   - `/dbcheck` で `SKU-APP` のINSERT経路が `200` / JSON で返ることも記録可能。
   - 失敗時でもCI自体は即死しないようにし、ログは必ず残す。

4. **証跡のアーティファクト化**

   - `actions/upload-artifact@v4` で `evidence/-.log` と `infra/20-alb/terraform.tfstate` を保存。
   - Smoke結果を「実際に到達・200応答・DB INSERTできた」物理ログとして持ち帰れるようにした。

5. **自動クリーンアップ**

   - ジョブ末尾で必ず `terraform destroy -auto-approve` を実行。
   - ALB / Target Group / SG / 一時Ingress をすべて破棄し、リーク・コスト残を防止。
   - destroy まで含めてワークフロー全体が成功状態で完走することを確認済み。

6. **変数とIAM権限の整備**

   - Terraform側に `ecs_tasks_sg_id` を追加し、`default = null` と `count = var.ecs_tasks_sg_id == null ? 0 : 1` で任意化。ローカル検証では未設定でも plan/apply が通る。CIでは Secrets から渡して有効化。
   - GitHub Actions で `PUBLIC_SUBNET_IDS`, `VPC_ID`, `ECS_TASK_SG_ID` などを runtime tfvars (`dev.auto.tfvars`) として書き出すフローを確立。
   - IAMロールには ALB/TG 周り (elasticloadbalancing系のDescribe/Modify/RegisterTargets 等)、ECS (ListTasks/DescribeTasks)、EC2 SG編集など最小限の権限を付与済み。

7. **CIジョブの成功確認**

   - 最新実行は `succeeded`。
   - 「作る → 疎通確認 → 証跡取得 → 破壊」が1本の workflow 内で成立した。

### 証跡

- GitHub Actions 実行結果のスクリーンショット
  (ジョブ `smoke` が `destroy` まで緑で完走している状態)

- `evidence/*_healthz.log`, `evidence/*_dbcheck.log`
  - ALB DNS に対して `curl -si /healthz` と `/dbcheck` を実行した生ログ
  - `/dbcheck` は RDS(PostgreSQL) に対する INSERT/UPSERT 経路 (`papyrus_schema.products` への SKU-APP) を通し、JSONを返すことを確認。

- `infra/20-alb/terraform.tfstate` のアーティファクト
  - CIがそのrunで実際に立てた ALB / Target Group / SG / 一時Ingress のリソースIDが入っている。監査証跡として利用可能。

- ECSタスク検出と `register-targets` のログ
  - RUNNINGタスクのプライベートIPを特定。
  - `private_ip:5000` を Target Group に一時登録。
  - `aws elbv2 wait target-in-service` が成功していることを記録。

これにより「Papyrus は RDS 付きの実稼働中 ECS タスクを CI が動的に拾い、ALB 経由で HTTP 200 を返す」ことを客観的に証明できる。

### ロールアウト

- `alb-smoke` ワークフローを1回走らせるだけで、ALB / SG / TG / ECS / RDS / Flask / Gunicorn / Secrets Manager / SSM / psycopg2 / INSERT 経路まで含めた実インフラのスモークテスト (ほぼE2E) が自動実行される。
- destroy が必須ステップなので、検証後にリソースも課金も残らない。
- `evidence/*.log` と `terraform.tfstate` がartifact化されるため、監査・技術ブログの裏付け資料にそのまま使える。

### 残課題

- [x] `/healthz` をDB非依存の軽量エンドポイントとして固定し、TargetGroupの `health_check.path` を `/healthz` に統一する (現状は `/` を流用しているケースがある)。
- [x] `/dbcheck` のJSONレスポンス (SKU-APPや inserted=true 等) をartifactとして恒常的に保存し、RDSへのINSERT証跡を明文化する。
- [ ] PG接続のSSL (`PGSSLMODE=require`) をタスク定義に組み込み、平常運用とCIの接続ポリシーを統一する。
- [x] Secrets / RDS ドリフト検知、FlaskのBlueprint未登録でタスクがExit 3する問題の再発防止など、起動前プリフライトの自動化は未統合。
- [ ] CloudWatchアラーム (タスクExit Code, メモリ圧迫, 将来のALB 5xxなど) はまだTerraform化していない。
- [x] README整備(2025-10-21以降分、`ECS_TASK_SG_ID` の注入方法など) が未反映。今後 `infra/20-alb/README.md` に反映する。

---

## 2025-10-24 → 2025-10-28

### 目的

Papyrus 環境において、ALB/TG の一時デプロイを自動化し、作成から破棄までを CI (GitHub Actions) 上で再現可能にすること。これにより「都度手でALBを作って/残骸を消し忘れて課金・名前衝突する」というリスクを排除し、証跡（DNS名/Target Group ARN等）を自動取得できる状態まで引き上げる。

### 主要変更

- `alb-smoke` ワークフローを整備し、以下が 1 ジョブ内で完結するようになった。

  - Terraform init/apply を実行し、Application Load Balancer / Target Group / Security Group をプロビジョニング
  - Terraform output から ALB DNS / Target Group ARN などを収集し、ジョブ内でエクスポート
  - 収集した出力をアーティファクトとして保存
  - 最後に Terraform destroy を必ず実行し、ALB/TG/SG を破棄してクリーンな状態に戻す
    （`destroy` ステップは `always` 扱いで、apply 中に失敗が出ても最終的に後片付けされる）

- dev.tfvars 相当の機密値（VPC ID / サブネットID / ECSタスクSGなど）はリポジトリに含めず、GitHub Actions 実行時に一時的にファイル生成する運用に変更。
  → 構成は IaC で再現可能、かつシークレットはリポジトリに残さない形に整理。

- IAM 権限を追加し、GitHub Actions 実行ロール（`ECRPowerUser` Assumeロール）に ALB/TG/Listener 周りのライフサイクル操作が許可されるようにした。
  具体的には以下のAPI権限を追加済み:

  - `elasticloadbalancing:CreateLoadBalancer` / `DeleteLoadBalancer` / `ModifyLoadBalancerAttributes` / `DescribeLoadBalancers` / `DescribeLoadBalancerAttributes`
  - `elasticloadbalancing:CreateTargetGroup` / `DeleteTargetGroup` / `ModifyTargetGroup` / `ModifyTargetGroupAttributes` / `DescribeTargetGroups` / `DescribeTargetGroupAttributes` / `DescribeTargetHealth` / `RegisterTargets` / `DeregisterTargets`
  - `elasticloadbalancing:CreateListener` / `DeleteListener` / `ModifyListener` / `DescribeListeners` / `DescribeListenerAttributes` / `ModifyListenerAttributes`
  - `elasticloadbalancing:AddTags` / `RemoveTags` / `DescribeTags`
  - `ec2:CreateSecurityGroup` / `DeleteSecurityGroup` / `AuthorizeSecurityGroupIngress` / `RevokeSecurityGroupIngress` / `AuthorizeSecurityGroupEgress` / `RevokeSecurityGroupEgress` / `DescribeSecurityGroups` / `DescribeSecurityGroupRules` / `DescribeVpcs` / `DescribeSubnets` / `DescribeNetworkInterfaces`

  これにより、前回まで発生していた以下の失敗を解消。

  - `DescribeListenerAttributes` 403 により Listener 作成後の属性参照で Terraform が落ちる
  - `ModifyLoadBalancerAttributes` / `ModifyTargetGroupAttributes` 403 により state 反映に失敗しゾンビ残留
  - ALB/TG/SG を作った途中で CI が停止し、次回 apply 時に `InvalidGroup.Duplicate` 等で衝突する

### 証跡

- GitHub Actions: `alb-smoke #9`

  - ステップ一覧:

    - `Setup Terraform`
    - `AWS creds` (AssumeRole 成功)
    - `Write tfvars (runtime only)`（機密を実行時だけ生成）
    - `Terraform apply (ALB/TG only)`
    - `Export outputs`（ALB DNS名 / Target Group ARN 等を取得）
    - `Save outputs artifact`（証跡をアーティファクト化）
    - `Terraform destroy (always)`（リソースを破棄し終了時はクリーン）
  - ジョブ全体のステータス: `succeeded`

- CloudShell 側のローカル証跡:

  - `*_alb_sg_leftover.json`

    - 残存していた `papyrus-alb-sg` の SecurityGroupId/VpcId を記録
  - `*_alb_sg_delete.log`

    - 上記 SG の削除ログ
  - `*_alb_sg_verify_gone.json`

    - `describe-security-groups` で `papyrus-alb-sg` が空配列になることを確認

- IAM 更新手順の記録:

  - `ECRPowerUser` ロールに対し、ELBv2 (ALB/TargetGroup/Listener) と SG 周りの Create/Describe/Modify/Delete/TAG 系アクションを付与したインラインポリシーを適用済み

### ロールアウト

- `alb-smoke` ワークフローを実行することで、Papyrus用の ALB / Target Group / リスナー / セキュリティグループを一時的に構築し、Terraformの `output` を保存したうえで、同じジョブの最後で確実に破棄できるようになった。
- これにより、手動オペレーション無しで「Papyrusの公開経路(ALB経由)を最低限のIaCで再現し、後片付けまで保証する」というスモークテストが日単位で再現可能になった。
- 破棄が `always` 指定になっているため、apply 中に失敗しても課金リソース（ALB/TG/SG）だけ残留する事故は基本的に抑止される。

### 残課題

- [x] ALB/TG 作成までは自動化済みだが、以下は未自動化:

  - ECSタスクを Target Group に一時登録する処理 (`register-targets`)
  - ALB経由で `/healthz` を叩き、200 OK を取得する疎通テストの自動実行と、そのレスポンス/ステータスの証跡化
  - ALB→ECS 経由の `/dbcheck` 呼び出しでアプリ側 INSERT (`SKU-APP`) が成功したログを取得・保管するステップ

    - 現時点では `/dbcheck` は Flask 側に追加済み、CLI経由の `SKU-CLI` INSERT は確認済みだが、ALB経由のアプリINSERT成功の証跡 (`200 OK` + JSON) はまだ未取得

- [ ] タスク定義側の恒久化対応:

  - `PGSSLMODE=require` を ECS タスク定義の環境変数として常設し、平文接続を防ぐ
  - `/healthz` (DB 非依存の軽量エンドポイント) をアプリに実装し、ALB Target Group のヘルスチェックパスに使えるようにする
    → これが入ると ALB/TG 側のヘルス確認もワークフローで自動化しやすくなる

- [ ] 監視/監査系:

  - `terraform apply` 時に CloudWatch Alarm (ECS Memory >80%, ALB 5xx%, TargetResponseTime p90 >1.5s 等) と SNS Topic を最小セットで同時に立て、証跡ログを取得するステップは未導入
  - Secrets Manager (`papyrus/prd/db`) の接続情報と RDS 実体の Endpoint/DB名 の diff チェックを CI のプリフライトに入れるのは未実装

- [x] ドキュメント:

  - `infra/20-alb` 用 README（「Runtime tfvars で apply/destroy する」「権限セットの最小要件」「ゾンビSGが残った場合の手動掃除手順」「出力アーティファクトの参照方法」）をまだ整理していない
    → 次回の Zenn/ポートフォリオ記事に載せるため、証跡ファイル名と一緒にまとめる必要あり

---

## 2025-10-21

### 目的

1. Papyrus の RDS スキーマを安全に投入し、アプリ/CLIの二系統で INSERT 0 1 を証跡化する。SSL必須・最小SGを維持したまま運用可能状態に上げる。
2. Papyrus のアプリ改修と再デプロイを通し、ECS/Fargate 上で /dbcheck を有効化。イメージを digest 固定で差し替え、ECS Exec を有効化したうえでルーティングを実機確認する。

### 主要変更

- `init.sql` を ECS Exec 経由で DRY-RUN → 本適用（BEGIN…ROLLBACK/COMMIT）
- 2系統の挿入検証:
  - CLI系: psycopg2 直で SKU-CLI を挿入
  - アプリ経由: 暫定 /dbcheck で SKU-APP を挿入
- RDS SG は 5432 inbound を ECSタスクSGのみ に統一（重複SG整理済）
- `papyrus/routes.py` と名前衝突しないよう `papyrus/blueprints/dbcheck.py` に移動、`__init__.py` で Blueprint 登録。
- 実行中タスクに対し Exec で URL マップ確認。`/dbcheck` ルートの搭載を確認。

### 証跡

- `*_schema_dryrun_exec.log`, `*_schema_apply_exec.log`
- `*_papyrus_psql_insert_cli.log`, `*_papyrus_psql_insert_app.log`
- `*_rds_sg_inbound_after.json`
- 実行中イメージ
  - `*_running_image.log`
- URLマップ
  - `*_flask_url_map_after.log`
- /dbcheck 叩き込みログ（prefix 探査スクリプト込み）
  - `*_papyrus_psql_insert_app.log`
- サービス更新・安定待ち関連（必要に応じて script ログに追記）

### ロールアウト

- `us-west-2`、サービス `papyrus-task-service`。外部公開無し、影響はタスク1本のみ。
- 認証は OIDC。Secrets papyrus/prd/db は RDS 実体と整合済み（host/port/user/dbname/password）。
- タスク定義: `papyrus-task:39` をサービス `papyrus-task-service` に適用済み。
- サービス状態: Desired=1 / Running=1 まで復旧。
- ALB 経由のヘルスチェックは未設定だが、アプリは `0.0.0.x:5000` で稼働し、`/dbcheck` が URL マップに載っていることを Exec で確認済み。

### 残課題

- [x] /healthz を軽量実装して将来の ALB/TG ヘルスに流用
- [x] /dbcheck の 200 実測とレスポンス保存(今日は URL マップまで。次回、/dbcheck 実行で 200 と JSON をログに残す)
- [x] CI の安全策
  - [x] コンテナ起動前テスト: `python -c "from papyrus import create_app; a=create_app(); print([r.rule for r in a.url_map.iter_rules()])"` を CI で回し、/dbcheck の存在を検知。
  - [x] ECS Exec 有効 のサービス設定 drift チェックを IaC 側に。
- [ ] `PGSSLMODE=require` をタスク定義で恒久化、可能なら sslrootcert 検証まで
- [x] CI プリフライト: Secrets と RDS 実体の `diff`、RDS エンドポイント変更検知
- [ ] CloudWatch Alarm（ECSメモリ/CPU、将来のALB 5xx/応答遅延）
  - [ ] 退出コード異常（Exit 3 など）と起動失敗の CloudWatch アラームを追加。
- [x] RDS 初期化フローの二段化
  - 既存の `init.sql` は本番データ扱い。DRYRUN と本適用をスクリプトで明確に分離し、Evidence を自動保存。


## 2025-10-08 → 2025-10-17

### 目的

ECSデプロイ時にロールバックが頻発する問題が発生。
Terraform記述のDB名、パスワードがSecretsと不整合を起こしていることが原因。
まず、Secrets不整合とSSL未指定による起動失敗を解消する。
次に、Papyrus を「DB接続で即死させずに」正常起動させ、VPC内からHTTP 200を確認する。

### 主要変更

- **ECS Exec有効化**: IAMロール修正、CloudShellからpsqlコマンド直叩きを有効化。
- **現行Secrets検証**: `papyrus/prd/db` の `database`/`password` をRDS実体と突き合わせて確認。
- **RDS再起動**: ParameterGroup反映確認のため再起動実施（`rds.force_ssl=1` 維持、ApplyType=dynamic確認）。
- **Secrets更新**: `papyrus/prd/db` を正値に上書き（`database=papyrus`、正しい`password`、既存の`host/port/username`）。
- **タスク更新**: サービスを `**force-new-deployment` で再デプロイ。必要に応じて `PGSSLMODE=require` をタスク定義へ付与し再登録。
- **VPC内疎通確認**: ECS Exec からアプリ直叩き。`/healthz` は未実装で404、`/` は200で生存判定OK。

### 証跡

- RDSパラメータ確認: `describe-db-parameters` で `rds.force_ssl=1 (dynamic)` を記録
- Secrets現値・更新:
  - `aws secretsmanager get-secret-value **secret-id papyrus/prd/db` 出力（更新前/更新後）

- ECSデプロイ/状態:
  - `aws ecs update-service **force-new-deployment` 実行ログ
  - `aws ecs wait services-stable` 完了
  - `aws ecs describe-services` で `desiredCount=1 / runningCount=1` を確認

- HTTP疎通（ECS Exec 内 Python）:
  - `/healthz -> 404 Not Found`（未実装のため想定内）
  - `/ -> 200 OK` 本文 `Welcome to Papyrus` を確認

### ロールアウト

- 対象: `us-west-2` の Papyrus（Fargate, cluster `papyrus-ecs-prd`, service `papyrus-task-service`）
- 影響範囲: 新リビジョンのタスク1本のみ。ALB未連携のため外部トラフィック影響なし。
- 認証: OIDC（既存設定）。Secrets/SSM読み取りは `papyrusTaskRole`。
- 結果: サービス安定（`services-stable`）、アプリHTTP 200をVPC内で確認。

### 残課題

- [x] **ヘルスエンドポイント実装**: `/healthz` をDB非依存で200返す軽量版で追加。将来ALB/TGのHCに流用。
- [ ] **ALB/TG連結の最小化**: `containerName=app`/`containerPort=5000` でターゲット登録。ALBアクセスログ先S3だけ先に用意。
- [x] **DB経由の実証**: 一時エンドポイント `/dbcheck` 等で `INSERT 0 1` をアプリ経由で実演し証跡化。
- [ ] **SSLの恒久化**: タスク定義に `PGSSLMODE=require` を常設。可能なら `sslrootcert=rds-combined-ca-bundle.pem` を同梱して検証強化。
- [x] **CIプリフライト**: デプロイ前に「SecretsとRDS実体のdiffチェック」をWorkflowに追加（`DBName/Endpoint/Port/Username`）。
- [ ] **監視**: CloudWatch Alarm（ECSメモリ、ALB 5xx%、TargetResponseTime）をPapyrus側にも適用。
- [x] **ドキュメント**: `infra/10-rds/README.md` に「手作業との差分・再起動時刻・ParameterGroup差分」を追記。

---

## 2025-09-10

*CloudTrail 90日分の証跡を取得し保全（CloudShell実施）*

### 目的

Papyrus関連の操作履歴をローカル/リポジトリで長期保全し、監査・インシデント解析に備える

### 実行環境

AWS CloudShell（アカウント: Papyrus 運用、リージョン: `us-west-2`）

### 実施内容

- `cloudtrail:LookupEvents` を用いて直近90日のイベントを **NDJSON**（1行1イベント）で全件取得
- 「Papyrus」関連のみをフィルタした派生ファイルも生成
- それぞれを gzip 圧縮しダウンロード、`docs/evidence/cloudtrail/` に保存

### 証跡

- `docs/evidence/cloudtrail/<timestamp>/cloudtrail_events_<UTC>.jsonl.gz`（全イベント）
- `docs/evidence/cloudtrail/<timestamp>/cloudtrail_events_<UTC>.papyrus.jsonl.gz`（Papyrus関連のみ）
