こんにちは。SRE部の庭野です。
今回はSRE部が運用管理しているプロジェクト依存関係の更新を自動化してくれるツールであるRenovateについて記事にしました。

手作業での更新確認が負担となっているプロジェクトに対し、Renovateが運用改善に役立つことを紹介します。

# Renovate とは？
Renovateは、ソフトウェアプロジェクトの依存関係を自動的に検知／更新するツールです。

ここで言う依存関係とは、私たちのプロジェクトが正しく動作するために必要な外部のコードやライブラリといったパッケージのことを意味します。

依存関係管理は、放置するとセキュリティリスクの増大やパフォーマンスの低下を引き起こす可能性があります。
Renovateを使用することで、これらの問題を効果的に防ぐことができます。
自動化されたプロセスにより、開発チームは新しい機能の追加やバグ修正に集中できるようになり、プロジェクトの健全性を維持しつつ、セキュリティを強化することが可能です。

# 主な機能
Renovateは自動的にプルリクエストを作成し、依存関係の更新を提案します。
また、更新を行うタイミングを細かく設定できるため、チームのワークフローに合わせて更新作業のスケジューリングが行えます。

# 導入概要
DMMではGitHub Cloudの他にGitHub Enterprise Server（以降GHES）も活用しています。
GitHub CloudのRenovateはGHESではそのままでは利用できず、Self Hostedとして設定する必要があります。
※本記事ではSelf Hosted型のRenovate Bot実行基盤の導入手順については省略します。

GHESでRenovateの利用を希望する場合、まず管理者が作成したRenovate Botをオーガニゼーションに追加する必要があります。
その後、任意のリポジトリ（もしくは全てのリポジトリ）でRenovateを有効にして実行対象に設定します。
<<イメージ画像を挿入する>>

これにより、Renovate Botが初めて実行されたとき、Renovate実行に必要なrenovate.jsonファイルを含む初期設定プルリクエストが自動的に作成されます。
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

# ベストプラクティスと注意点
Renovateを最大限に活用するためには、自動更新を適用する依存関係を選ぶことが重要です。
不必要に多くの自動更新を設定すると、それぞれの更新について確認・テストが必要となり、管理が大変になるからです。

# まとめ
Renovateを使うことで、プロジェクトの依存関係を簡単に最新の状態に保つことができます。
これにより、セキュリティを強化し、開発の効率を高めることが期待できます。

この記事を通じて、Renovateがどのようにプロジェクトの依存関係管理を効率化し、セキュリティリスクを軽減するのかを理解していただけたと思います。
自動化により時間の節約が可能となり、開発チームはより重要なタスクに集中できます。
次のステップとして、自身のプロジェクトにRenovateを導入してみることをお勧めします。

Renovateを導入することで、効率的な依存関係管理とセキュリティの向上が可能になります。
