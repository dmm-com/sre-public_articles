こんにちは。ITインフラ本部 SRE部の庭野です。
今回は、私たちSRE部で運用管理しているプロジェクトの依存関係更新作業を自動化するツール、Renovateについて紹介します。

多くのプロジェクトでは、依存関係の更新が手作業で行われ、大きな負担となっています。
Renovateを活用することにより、この更新プロセスを自動化し、開発チームがより創造的な作業に集中できるようになります。

※この記事では、Renovateのバージョン37.279.0を前提に説明しています。

# Renovate とは？
Renovateは、ソフトウェアプロジェクトの依存関係を自動的に検知／更新するツールです。
※公式サイト: https://docs.renovatebot.com/

ここで言う依存関係とは、私たちのプロジェクトが正しく動作するために必要な外部のコードやライブラリといったパッケージのことを意味します。

依存関係の適切な管理を怠ると、使用されている外部ライブラリが古くなり、セキュリティの脆弱性やパフォーマンスの問題を引き起こす可能性があります。
日々のアップデートは手間がかかり、しばしば見過ごされがちです。
Renovateを使用することで、これらのライブラリの継続的な更新が自動化され、セキュリティリスクの増大を防ぎながら、手間を省くことができます。
自動化されたプロセスにより、開発チームは新しい機能の追加やバグ修正に集中できるようになり、プロジェクトの健全性を維持しつつ、セキュリティを強化することが可能です。

# 主な機能
Renovateは自動的にプルリクエストを作成し、依存関係の更新を提案します。
また、更新を行うタイミングを細かく設定できるため、チームのワークフローに合わせて更新作業のスケジューリングが行えます。

# 導入概要
DMMではGitHub Cloudの他にGitHub Enterprise Server（以降GHES）も活用しています。
GitHub CloudのRenovateはGHESではそのままでは利用できず、Self Hostedとして設定する必要があります。
本記事ではRenovate Botをセットアップする具体的な手順には触れず、Renovateが提供する主要な機能とそのメリットに焦点を当てて説明します。

GHESでRenovateを利用する場合、事前に用意したRenovate Botをオーガニゼーションに追加する必要があります。
Renovateを有効にしたいリポジトリで、以下のようにRenovate Botからのアクセスを許可します。

![repository-access.png](repository-access.png)

これにより、Renovate Botが初めて実行されたとき、Renovateの実行に必要なrenovate.jsonファイルを含む初期設定プルリクエストが自動的に作成されます。
このプルリクエストをマージすることで、Renovateによる基本的な依存関係の更新設定が完了します。

* 作成されたプルリクエスト

![pr.png](pr.png)

* プルリクエストの内容

![pr-detail.png](pr-detail.png)

ここまでのプロセスはシンプルで、わずか数ステップで完了します。

次に、実際にRenovateをどのように設定して使用するかを見てみましょう。

# 実践的な使用例
初期設定のrenovate.jsonのままでは期待した動作にならないことがほとんどです。

```
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json"
}
```

以下は、Renovateのカスタマイズが必要となる一般的なシナリオです。

- パッケージがリリースされてから指定日数はプルリクエストを作成せずに様子を見たい
- 指定日にだけ依存関係を更新したい
- 一度のRenovate実行で作成されるプルリクエスト数を制限したい
- 一度に保持できるプルリクエスト数を制限したい
- プルリクエスト作成時に自動的に担当者アサイン／レビュアーアサインを行いたい

こういった場合は、以下のようにrenovate.jsonをカスタマイズすることで、これらの要望に応えることができます。

```
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:best-practices",
    ":label(renovate)",
    ":timezone(Asia/Tokyo)"
  ],
  "configMigration": true,
  "minimumReleaseAge": "7 days",
  "prHourlyLimit": 5,
  "prConcurrentLimit": 10,
  "schedule": [
    "after 10am and before 5pm every weekday"
  ],
  "assignees": [
    "<user-name>"
  ],
  "reviewers": [
    "<user-name> or <team-name>"
  ]
}
```

それぞれのパラメータについて確認してみましょう。

| パラメータ | 説明 |
| --- | --- |
| `$schema` | スキーマのURL。 |
| `extends` | ベースとなる設定。 |
| `configMigration` | 設定の移行を有効にするかどうか。 |
| `minimumReleaseAge` | パッケージがリリースされてから指定日数はプルリクエストを作成せずに様子を見る設定。 |
| `prHourlyLimit` | 一度のRenovate実行で作成されるプルリクエスト数を制限する設定。0は無制限。 |
| `prConcurrentLimit` | 一度に保持できるプルリクエスト数を制限する設定。0は無制限。 |
| `schedule` | 指定日にだけ依存関係を更新する設定。 |
| `assignees` | プルリクエスト作成時に自動的に担当者アサインを行う設定。 |
| `reviewers` | プルリクエスト作成時に自動的にレビュアーアサインを行う設定。 |

これらを設定することで期待する動作が実現できます。

さらに応用的な使用例として、以下のような検出も行うことが可能です。

- パッケージやディレクトリ単位でグルーピング更新
  - サンプルコード
```
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "packageRules": [
    {
      "matchManagers": [
        "terraform"
      ],
      "additionalBranchPrefix": "{{packageFileDir}}-",
      "commitMessageSuffix": "({{packageFileDir}})",
      "groupName": "terraform",
      "addLabels": [
        "terraform"
      ]
    }
  ]
}
```
- Amazon ECRで管理しているプライベートリポジトリ内コンテナのイメージタグ更新
  - サンプルコード
```
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "hostRules": [
    {
      "hostType": "docker",
      "matchHost": "<AWS Account ID>.dkr.ecr.<Region Name>.amazonaws.com",
      "username": "AKIAXXXXXXXXXXXXXXXX",
      "encrypted": {
        "password": "wcFMA/XXXXXXXXXXX..."
      }
    }
  ]
}
```
- GitHub Packageのバージョン更新
  - サンプルコード
```
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "hostRules": [
    {
      "hostType": "npm",
      "matchHost": "https://npm.pkg.github.com/",
      "encrypted": {
        "token": "wcFMA/XXXXXXXXXXX..."
      }
    }
  ],
  "npmrc": "@<Organization Name>:registry=https://npm.pkg.github.com/"
}
```

パラメータの詳細については、[公式ドキュメント](https://docs.renovatebot.com/config-overview/)を参照してください。
また、暗号化（encrypted）については[こちら](https://github.com/renovatebot/renovate/blob/main/docs/usage/getting-started/private-packages.md#adminbot-config-vs-userrepository-config-for-self-hosted-users)を参照してください。

# ベストプラクティスと注意点
Renovateを最大限に活用するためには、どの依存関係に自動更新を適用するか慎重に選択することが重要です。
不必要に多くの自動更新を設定すると、それぞれの更新について確認・テストが必要となり、管理が大変になるからです。

# まとめ
Renovateを使うことで、プロジェクトの依存関係を簡単に最新の状態に保つことができます。
これにより、セキュリティを強化し、開発の効率を高めることが期待できます。

この記事を通じて、Renovateがどのようにプロジェクトの依存関係を最新の状態に保ち、セキュリティリスクを軽減し、開発の効率を高めるかを理解していただけたと思います。
自動化により時間を節約し、開発チームがより重要なタスクに集中できるようになるため、Renovateの積極的な活用をお勧めします。
