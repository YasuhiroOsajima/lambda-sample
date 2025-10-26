# Lambda 関数のエラーをメールで通知する

## 1) Lambda のコードとスクリプト

ディレクトリ構成（例）

```
project/
├─ lambda/
│  ├─ handler.py
│  └─ script.sh
└─ terraform/   （後述の IaC を置く）
```

`lambda/script.sh`（失敗・成功を切り替えやすい例）

```
#!/usr/bin/env bash
set -Eeuo pipefail

# ここに本来の処理を書く
echo "[script] doing something..."

# 疑似的に失敗させたいときは以下を有効化
# exit 42

# 正常終了
exit 0
```

lambda/handler.py（Python 3.12 例）

```
import subprocess
import logging
import os

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def run_script():
    script_path = os.path.join(os.path.dirname(__file__), "script.sh")
    # bash -e: エラーで即時終了
    proc = subprocess.run(
        ["bash", "-e", script_path],
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        text=True,
        check=False,         # 自前で returncode を判定
    )
    # 標準出力をそのままログへ
    if proc.stdout:
        for line in proc.stdout.splitlines():
            logger.info(line)

    if proc.returncode != 0:
        # 要件どおり「ERROR」を含む行を必ず出す（誤検知を減らすため "ERROR|" を接頭に）
        logger.error(f"ERROR|script_failed returncode={proc.returncode}")
        # ここで例外を投げる/投げないはお好み（再試行したいなら投げる）
        raise RuntimeError(f"script failed with code {proc.returncode}")

def lambda_handler(event, context):
    logger.info("handler start")
    run_script()
    logger.info("handler end")
    return {"ok": True}
```

ポイント

- 失敗時に 必ず ERROR|script_failed ... の固定フォーマットを出すことでメトリクスフィルタの誤検知を防ぎます。
- 例外を再スローすると、Lambda の再試行（同期呼び出しなら例外、非同期なら自動再試行／DLQ/再配信ポリシーの連携等）が使えます。

## 2) インフラ（Terraform 例）

下記を terraform/ に配置します。メール通知先は変数で受け取ります。

variables.tf

```
variable "project_name" {
  type    = string
  default = "lambda-script-monitor"
}

variable "notification_email" {
  type = string
}
```

main.tf

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.4"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

# --- Lambda パッケージ化 ---
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/../lambda"
  output_path = "${path.module}/build/lambda.zip"
}

# --- IAM ロール & ポリシー（CloudWatch へログ出力） ---
resource "aws_iam_role" "lambda_role" {
  name               = "${var.project_name}-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume.json
}

data "aws_iam_policy_document" "lambda_assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role_policy_attachment" "basic_exec" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# --- Lambda 本体 ---
resource "aws_lambda_function" "fn" {
  function_name = "${var.project_name}-fn"
  role          = aws_iam_role.lambda_role.arn
  handler       = "handler.lambda_handler"
  runtime       = "python3.12"
  filename      = data.archive_file.lambda_zip.output_path
  architectures = ["arm64"]
  timeout       = 30
  memory_size   = 256
  environment {
    variables = {
      LOG_LEVEL = "INFO"
    }
  }
}

# --- ロググループ（明示して保持期間など管理） ---
resource "aws_cloudwatch_log_group" "lg" {
  name              = "/aws/lambda/${aws_lambda_function.fn.function_name}"
  retention_in_days = 30
}

# --- "ERROR|" をカウントするメトリクスフィルタ ---
resource "aws_cloudwatch_log_metric_filter" "error_filter" {
  name           = "${var.project_name}-error-filter"
  log_group_name = aws_cloudwatch_log_group.lg.name

  # フィルタパターン：固定の "ERROR|" を含む行を検出
  pattern = "\"ERROR|\""

  metric_transformation {
    name      = "${var.project_name}-error-count"
    namespace = "Custom/LambdaLog"
    value     = "1"
  }
}

# --- 通知用 SNS ---
resource "aws_sns_topic" "alert_topic" {
  name = "${var.project_name}-alerts"
}

resource "aws_sns_topic_subscription" "email_sub" {
  topic_arn = aws_sns_topic.alert_topic.arn
  protocol  = "email"
  endpoint  = var.notification_email
}

# --- CloudWatch Alarm（1データポイントでアラームへ） ---
resource "aws_cloudwatch_metric_alarm" "error_alarm" {
  alarm_name          = "${var.project_name}-error-alarm"
  alarm_description   = "Lambda log contains ERROR|"
  namespace           = "Custom/LambdaLog"
  metric_name         = "${var.project_name}-error-count"
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 1
  threshold           = 1
  comparison_operator = "GreaterThanOrEqualToThreshold"
  treat_missing_data  = "notBreaching"

  alarm_actions = [aws_sns_topic.alert_topic.arn]
  ok_actions    = [aws_sns_topic.alert_topic.arn]
}

output "lambda_name" {
  value = aws_lambda_function.fn.function_name
}
output "sns_topic" {
  value = aws_sns_topic.alert_topic.arn
}
```

補足
メトリクスフィルタはログが出たときのみ値 1 を発行します。
アラームは「合計 ≥ 1（1 分間）」で発報。誤検知を避けたい場合はさらに ERROR|script_failed などに絞っても OK。

## 3) デプロイ手順（概略）

```
# 1) 依存ファイルに実行権限を付与
chmod +x ./lambda/script.sh

# 2) Terraform
cd terraform
terraform init
terraform apply -auto-approve -var="notification_email=あなたのメールアドレス"
```

初回、SNS のメール購読に 確認メール が届くので Confirm してください（購読が未確認だと通知が届きません）。

## 4) 動作確認（失敗 → ERROR → メール）

- script.sh の exit 42 を有効にして保存（意図的に失敗させる）。
- Lambda を手動実行（テストイベントは空で可）。
- CloudWatch Logs に ERROR|script_failed ... が出力され、メトリクスフィルタが検出。
- アラームが発火し、SNS からメールが届く。
- 正常系確認：exit 0 に戻して実行 → ERROR| は出ず、通知も来ない。

## 5) 運用上のベストプラクティス

- 固定フォーマット：`ERROR|code=...` のように接頭辞を固定し、アラームの対象を明確化。
- 冪等性：スクリプトは同一入力で同じ結果になるよう設計（再試行に耐える）。
- 再試行・死活監視：非同期呼び出しなら DLQ / EventBridge リトライポリシー、同期呼び出しなら呼び出し元側で再試行を。
- 過検知回避：通常ログで「error」という単語を出さず、必ず大文字の ERROR| のみを通知キーに。
- 権限最小化：Lambda 実行ロールは基本ポリシーのみ。S3 等を使う場合は必要最小の追加ポリシーを付与。
