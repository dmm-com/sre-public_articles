# ECSタスクの停止ログを残しましょう

## はじめに

こんにちは。ITインフラ本部 SRE部のシュウです。
本記事では、DMMブックスで実際に発生した障害の恒久対応の一環として、ECSタスクの停止ログを記録する方法についてご紹介します。

## 背景

DMMブックスでは、トップページの新着枠に表示される内容をECSの「スケジュールされたタスク」機能を利用し、1分ごとにデータを更新しています。
この処理が正常に動作しないと、新着枠の内容が更新されず、古い情報が表示されたままとなってしまいます。

## 障害の発生原因

コスト削減施策の一環として、アプリケーションを稼働させているECS FargateをAWS Gravitonに移行することを進めています。（2025年1月時点では、東京リージョンで約20%のコスト削減が可能です。）

AWS Gravitonに移行することでコスト削減が可能ですが、CPUアーキテクチャがx86_64からarm64に変更されるため、アプリケーションもそれに対応する必要があります。

ECS Fargate上で稼働しているアプリケーション自体は事前にarm64アーキテクチャに対応させていたため、問題なくAWS Gravitonへ切り替えられました。

しかし、アプリケーションと同じECRのイメージを使用している「新着枠を更新するためのバッチ」に対する考慮が漏れていたため、バッチ用のECSタスク定義の変更が行われませんでした。その結果、x86_64アーキテクチャの環境上でarm64アーキテクチャ用にビルドされたイメージがデプロイされる形となりました。

このため、インフラとイメージのCPUアーキテクチャが一致せず、ECSタスクが起動できずに再起動を繰り返す事象が発生しました。

## 振り返り
ポストモーテムを作成し、関係者と読み合わせを行った結果、ECSタスクが起動できなかったことにすぐに気付けなかったことが問題の一つとして挙げられました。

ECSタスクがCloudWatch Logsに出力するログをトリガーにしてアラートを設定することも可能ですが、「ECRに必要なイメージが存在しない」「ECSタスクの起動に必要な権限が不足している」「参照しているパラメータが存在しない」など、ECSタスクが起動する前の段階でエラーが発生する場合もあります。その場合、そもそもログが出力されないため、アラートを設定することができません。

したがって、アプリケーションのログだけでなく、ECSタスク自体の停止ログを記録することが重要であると考えました。

## ECSタスクの停止ログをアラートに設定しよう
AWSコンソールから停止したタスクの詳細ページを開くと、以下の画像のように停止理由が表示されます。しかし、この情報はデフォルトでは保存されず、時間が経過すると消えてしまいます。

この停止理由を記録し、アラートを設定することで、万が一タスクが停止した際もその理由を把握することが可能になります。

![ecs-task-error-sample-01.png](./img/ecs-task-error-sample-01.png)

## NewRelic & DataDog にECSタスクの停止ログを送信する
ECSタスクのステータス変更はEventBridgeによって検知できるため、EventBridgeからKinesis Firehoseへ送信し、NewRelicやDataDogに停止ログを送信することが可能です。

![infra.png](./img/infra.png)

## Terraformを使った実装例
### Kinesis Firehose 用のIAMロールの作成
```terraform
resource "aws_iam_role" "firehose" {
  name = "ecs-task-stop-log-firehose-role"
  path = "/service-role/"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "firehose.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })
}

data "aws_iam_policy_document" "firehose" {
  statement {
    actions = [
      "s3:*",
    ]
    resources = [
      "arn:aws:s3:::<ログ保存用のS3バケット>",
      "arn:aws:s3:::<ログ保存用のS3バケット>/*",
    ]
  }
}

resource "aws_iam_role_policy" "firehose" {
  policy = data.aws_iam_policy_document.firehose.json
  role   = aws_iam_role.firehose.name
}
```
### Event Bridge 用のIAMロールの作成
```terraform
resource "aws_iam_role" "event_bridge" {
  name = "event-bridge-to-firehose-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "events.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })
}

data "aws_iam_policy_document" "event_bridge" {
  statement {
    actions   = ["firehose:PutRecord", "firehose:PutRecordBatch"]
    resources = [aws_kinesis_firehose_delivery_stream.ecs_task_stop_log.arn]
  }
}

resource "aws_iam_role_policy" "event_bridge" {
  role   = aws_iam_role.event_bridge.name
  policy = data.aws_iam_policy_document.event_bridge.json
}
```
### Kinesis Firehose の作成
- NewRelic
  - ログ転送用エンドポイント: `https://aws-api.newrelic.com/firehose/v1`
- DataDog
  - ログ転送用エンドポイント: `https://aws-kinesis-http-intake.logs.datadoghq.com/v1/input`
```terraform
resource "aws_kinesis_firehose_delivery_stream" "ecs_task_stop_log" {
  name = "ecs-task-stop-log"

  destination = "http_endpoint"

  http_endpoint_configuration {
    url                = "<NewRelic or DataDog のログ転送用エンドポイント>"
    name               = "<NewRelic or DataDog>"
    access_key         = "<NewRelic or DataDog のAPIキー>"
    buffering_size     = 1
    buffering_interval = 60
    role_arn           = aws_iam_role.firehose.arn
    s3_backup_mode     = "AllData"

    s3_configuration {
      role_arn           = aws_iam_role.firehose.arn
      bucket_arn         = "arn:aws:s3:::<ログ保存用のS3バケット>"
      prefix             = "ecs-task-stop-log/"
      buffering_size     = 1
      buffering_interval = 60
      compression_format = "GZIP"
    }

    request_configuration {
      content_encoding = "GZIP"
    }
  }
}
```
### EventBridge の作成
```terraform
resource "aws_cloudwatch_event_rule" "ecs_task_stop_rule" {
  name        = "ecs-task-stop-rule"
  description = "Capture ECS task stop events"
  event_pattern = jsonencode({
    source      = ["aws.ecs"]
    detail-type = ["ECS Task State Change"]
    detail = {
      lastStatus = ["STOPPED"]
    }
  })
}

resource "aws_cloudwatch_event_target" "ecs_task_stop_log" {
  rule     = aws_cloudwatch_event_rule.ecs_task_stop_rule.name
  arn      = aws_kinesis_firehose_delivery_stream.ecs_task_stop_log.arn
  role_arn = aws_iam_role.event_bridge.arn
}
```
