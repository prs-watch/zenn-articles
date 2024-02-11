---
title: "DependabotがActions Secretsにアクセスしてくれなかったことへの対策"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dependabot", "github"]
published: true
---

趣味アプリのリポジトリにDependabotを組み込んだのですが、Dependabotが作成したPRで実行されたActionsにて、Actions Secretsが読まれずにエラー落ちしていました。

https://docs.github.com/ja/code-security/dependabot/working-with-dependabot/automating-dependabot-with-github-actions#accessing-secrets

> Dependabot イベントでワークフローがトリガーされる場合、ワークフローで使用できるシークレットは Dependabot シークレットのみです。 GitHub Actions シークレットは使用できません。 

と言うことで、対策を検討したので簡単にログメモを展開します。

# 主な対策一覧

インターネットの海をサーフィンしたり、ChatGPTと壁打ちをした限りでは、下記4種の対策から吟味、との形になりました。

## Dependabot Secretsへの登録

https://docs.github.com/ja/code-security/dependabot/working-with-dependabot/configuring-access-to-private-registries-for-dependabot#storing-credentials-for-dependabot-to-use

恐らく本体が一番推奨しているであろう対策。Botとしてリポジトリ管理のActions Secretsアクセスは許容出来ないが、Dependabotが覗ける特定のSecretsを用意することで、Actionsキック時のエラー落ちを防止する感じです。

正面案であるものの、ActionsとDependabotでSecretsを二重管理することが辛いので、対案が無ければこれにしようかなー、で下記の他対策を確認していきました。

## Environment Secretsへの登録

https://docs.github.com/ja/actions/deployment/targeting-different-environments/using-environments-for-deployment

今回のリポジトリはCI/CDのタイミングで環境分離をしてあれこれをする意味が無い（本番デプロイはNetlify側で行っているので、GitHub側はテスト用のビルドだけ）のですが、こちらは環境に閉じたアクセスとしてDependabot経由でもSecretsを読み取れます。

GitHubのブランチ戦略的な意味での環境利用との文脈では無いですが、Dependabot Secretsを入れることでの二重管理問題に対処出来る上、環境アクセスや承認フローを組み込んだ堅実なPR管理が出来るので、**結論この対策に落ち着きました**。

## GitHub Actionsのイベントトリガーを `pull_request_target` にする

https://docs.github.com/ja/actions/using-workflows/events-that-trigger-workflows#pull_request_target

インターネットの海をサーフィンする中で一番見た対策。`pull_request` との違いは実行コンテキストで、こちらはブランチ側のコンテキスト、つまりマージ元ブランチ側のGitHub Actions定義が採用される一方で、`pull_request_target` はベースリポジトリの定義が利用されます。ベースリポジトリコンテキストなのでリポジトリ管理のSecretsにもアクセスが出来、Dependabot経由のPRでもSecretsを読んで実行が出来ます。

https://engineering.mercari.com/blog/entry/20230609-github-actions-guideline/

ただし、forkされた先からのPRも含め外部からのPRを受け入れて実行されるため、ビルド対象のコードに攻撃を仕込んでActionsを走らされた場合にセキュリティ懸念がある、と言われています。これでも良かったのですが、制御の面ではEnvironment Secretsの方が良さそうだったので、こちらは断念。

## Actionを手動実行する運用に切替

これはあんまり検討したい手段では無かったので早々にカットしました。実際、自動化はなるべくであれば損ないたくないところです。

# 結論

DependabotのSecrets問題はいくつか対策手段があり、私の場合は比較的がちがちに制御を入れたEnvironment Secretsの利用に舵を切りました。Dependabotのホストは本営が行っているのだから、このあたりは柔軟にやってほしいのですが、、

とは言え、調べる中でちょっとGitHubのセキュリティ設計？の一端を知ることが出来たので、ひとまずはポジティブな気持ちです。ただ、事象にする対策としてこれが良いのか、は微妙な気持ちも若干残存してもやもやしているため、ベストプラクティスあるぞ！とのコメントがあれば是非頂きたい次第です。