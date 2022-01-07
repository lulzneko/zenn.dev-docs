---
title: "GitHub Actions でコミットログからプルリクにラベル貼り"
emoji: "🏷️"
type: "tech"
topics: [ cicd, githubactions, github ]
published: false
announce:
  記事を書きました
  プルリクエストにコミットメッセージからラベルを自動的に付与する GitHub Actions のスクリプトを紹介。

  以前紹介した lerna-changelog を使っている環境では素晴らしい効果を発揮！！

  ⇒ GitHub Actions でコミットログからプルリクエストのラベルを貼る
  https://zenn.dev/lulzneko/articles/labeling-pr-from-commitlog-with-githubactions
---

## はじめに
[lerna-changelog](https://github.com/lerna/lerna-changelog) を使うと CHANGELOG.md をプルリクエストから生成できます。

CHANGELOG.md 生成時に変更ログの分類やタイトルが不適切であったことに気づいた場合、プルリクエストを修正することで対応できます。

これはコミットログを使う方法に比べて柔軟です。いっぽうで [Conventional Commits](https://www.conventionalcommits.org/) などにより適切なコミットログを付ける運用をしている場合に、プルリクエストへ同じ内容のラベルを貼るという二度手間が発生してしまいます。これは GitHub Actions の CI/CD へ、ひと手間はさむことで自動ラベリングすることにより解決できます。

lerna-changelog と GitHub Actions によるコミットログからの自動ラベル付けを紹介します。

### 解決したい課題
- プルリクエストのラベル付けに手間が発生している
- プルリクエストのラベル運用が適切に回っていない

### 本記事の前提
- [lerna-changelog でプルリクからリリースノート生成する](https://zenn.dev/lulzneko/articles/generate-release-notes-by-pr-with-lerna-changelog) の記事の環境とします
- ソースコードの変更をプルリクエスト・ベースで行っているものとします

### 動作環境
本記事は以下の環境で動作を確認しています。
- GitHub Actions Ubuntu


## プルリクエストのラベル定義
プルリクエストに付与するラベルは lerna-changelog でも使用します。package.json に GitHub リポジトリのラベルとの対応を設定します。

またコミットログの Prefix には、この package.json の定義のキーの「Type: 」を除いた「Feature」などの文字列を小文字にしたもの「feature」などを使います。コミットログを Conventional Commits などに合わせる場合は、それをラベルの文字列にします。
※ 完全一致である必要はなく GitHub Actions のスクリプトでマッチできれば大丈夫です

今回は以下のように定義し GitHub のリポジトリへも [ラベルを作成](https://docs.github.com/ja/issues/using-labels-and-milestones-to-track-work/managing-labels) しました。
```json:package.json
{
  "...": "省略",
  "changelog": {
    "labels": {
      "Type: Breaking": "💥 Breaking Change",
      "Type: Feature": "🎉 New Feature",
      "Type: Enhancement": "🚀 Enhancement",
      "Type: BugFix": "💉 Bug Fix",
      "Type: Deprecated": "⚠️ Deprecated",
      "Type: Docs": "📝 Documentation",
      "Type: Refactoring": "✨ Refactoring",
      "Type: Testing": "✅ Testing",
      "Type: Build": "🛠️ Build",
      "Type: Dependency": "📦 Dependency"
    }
  }
}
```


## ラベルを設定する Actions
プルリクエストにラベルを貼る Actions なので `pull_request` のイベントをトリガーとします。
作成時に付与できれば良いと考えるので `types: [ opened ]` としています。
※ 後述のラベルチェックで `type` が増えるので `if` で `opened` に限定しています

```yaml
on:
  pull_request:
    types: [ opened ]

jobs:
  add_labels:
    if: github.event.action == 'opened'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/github-script@v4.0
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const ret = await github.issues.listLabelsForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo
            });
            const definitions = ret.data
              .filter(value => value.name
              .includes(': '))
              .map(value => value.name.split(':')[1].toLowerCase().trim());

            const commits = await github.pulls.listCommits({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.payload.pull_request.number
            });
            const chunk = Array.from(new Set(commits.data
              .map(data => data.commit.message)
              .filter(msg => msg.includes(': '))
              .map(msg => msg.startsWith('chore: release') ? msg : msg.split(': ')[0])));

            const prefixes = chunk.map(value => {
              if (value === 'fix') { return 'bugfix'; }
              if (value.startsWith('chore(deps')) { return 'dependency'; }
              if (value.startsWith('chore: release')) { return 'release'; }
              return value;
            }).filter(value => 4 <= value.length);

            const labels = definitions
              .filter(definition => prefixes.filter(prefix => definition.startsWith(prefix)).length !== 0)
              .map(value => `type: ${value}`);

            if (labels && labels.length !== 0) {
              github.issues.addLabels({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.pull_request.number,
                labels
              });
            }
```

まず、リポジトリに定義されている全ラベルを取得します。
全ラベルから今回定義のラベルに絞り込み、ラベル名のみを小文字化、定義リストを作ります。
e.g. 「Type: Feature」などの今回定義の全ラベルを取得し「feature」のようなリストを作成

次に、プルリクエストにつく全コミットを取得します。
コミットメッセージから Prefix を取り出し一時リストを作成します。
e.g. 「feat: send mail when status changes」などのログから「feat」のようなリストを作成

続いて、コミットメッセージ Prefix の一時リストをラベルにマッチする Prefix へ変換します。
コミットメッセージとラベルが完全一致しないものを調整しラベルの文字列へ合わせます。

最後に、コミットメッセージから変換したラベルリストと、リポジトリから取得したラベルの定義リストからマッチングしたものをプルリクエストのラベルとして設定します。


## ラベルをチェックする Actions
プルリクエストにラベルを自動付与できるようになりましたが、コミットメッセージの Prefix が Actions の処理でマッチしないとラベルなしとなってしまいます。

lerna-changelog を使うからには適切なラベルが付けられるようにしたいものです。そのためプルリクエストにラベルが設定されていない場合はエラーとする GitHub Actions もしかけます。

プルリクエストイベントから、プルリクエスト自体の操作とラベルの操作に反応させるため `types: [ opened, synchronize, reopened, labeled, unlabeled ]` としています。またラベル付与ジョブが終わった後でないと付与前にエラーを返してしまうので `needs: [ add_labels ]`、またエラーがあってもチェックはしたいので `if: always()` としています。

```yaml
on:
  pull_request:
    types: [ opened, synchronize, reopened, labeled, unlabeled ]

jobs:
  check_labels:
    needs: [ add_labels ]
    if: always()
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v2.3.4
      - uses: actions/github-script@v4.0
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const { changelog } = require('./package.json');

            const ret = await github.issues.listLabelsOnIssue({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.payload.pull_request.number
            });
            const labels = new Set(ret.data.map(label => label.name));

            const definitions = new Set(Object.keys(changelog.labels));
            if (ret.data.length === 0 || new Set([...labels].filter(label => (definitions.has(label)))).size === 0) {
              let body = '### Labels for Release Note\n'
              body += '**[ERROR]**  \n';
              body += '- リリースノートの自動生成のためにラベルを付けてください。\n\n';
              body += 'The available labels are as follows:\n'
              body += '| Label | Release Note |   | Label | Release Note |\n'
              body += '|:------|:-------------|:-:|:------|:-------------|\n'

              const entries = Object.entries(changelog.labels)
              for (let i = 0; i < entries.length; i++) {
                body += `| ${entries[i][0]} | ${entries[i][1]} |`;
                body += entries[++i] ? `| ${entries[i][0]} | ${entries[i][1]} |\n` : '|||\n';
              }

              github.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.pull_request.number,
                body
              });
              return core.setFailed('No label has been set.');
            }
```

まず、設定すべきラベルのリストを `package.json` の lerna-changelog 定義から取得します。

次に、プルリクエストに設定されている全ラベルを取得します。

これらのリストを突き合わせてマッチするものが１つもない場合はエラー・メッセージをプルリクエストにコメントし、`return core.setFailed();` でエラー終了します。
※ 例では利用できるラベルをコメントするため複雑ですがシンプルなメッセージもよいでしょう


## ご参考
実際に使用しているラベルの対応表を載せておきます。
本記事は [サービスを Ship.js でプルリク･ベースにリリースする](https://zenn.dev/lulzneko/articles/using-shipjs-to-make-service-releases-pr-based) の流れで、サービスのリリースを前提としており用途や説明が少し特殊かもしれませんが、なにかの参考になれば幸いです。

| リリースノート      | Lv.    | 色                                                       | GitHub ラベル     | コミット接頭辞   | 説明                                 |
|:--------------------|:-------|:---------------------------------------------------------|-------------------|:-----------------|:-------------------------------------|
| 💥 Breaking Change | major  | ![#3E1E00](https://via.placeholder.com/15/3E1E00?text=+) | Type: Breaking    | (N/A)            | 提供済み機能の互換のない破壊的変更   |
| 🎉 New Feature     | minor  | ![#224B8F](https://via.placeholder.com/15/224B8F?text=+) | Type: Feature     | feature:         | API 追加などの新機能提供など         |
| 🚀 Enhancement     | minor  | ![#0071B0](https://via.placeholder.com/15/0071B0?text=+) | Type: Enhancement | enhance:         | 既存の提供機能に対する機能の拡張など |
| 💉 Bug Fix         | patch  | ![#E7001D](https://via.placeholder.com/15/E7001D?text=+) | Type: BugFix      | fix:             | 既存の提供機能に対する不具合修正     |
| ⚠️ Deprecated      | patch  | ![#AD002D](https://via.placeholder.com/15/AD002D?text=+) | Type: Deprecated  | deprecated:      | 既存の提供機能に対する非推奨化       |
| 📝 Documentation   | patch  | ![#737373](https://via.placeholder.com/15/737373?text=+) | Type: Docs        | docs:            | ドキュメントの追加や修正             |
| ✨ Refactoring     | patch  | ![#008D56](https://via.placeholder.com/15/008D56?text=+) | Type: Refactoring | refactor:        | 内部バグ修正や機能の追加ではない変更 |
| ✅ Testing         | patch  | ![#006428](https://via.placeholder.com/15/006428?text=+) | Type: Testing     | test:            | テストの追加や既存のテストの修正     |
| 🛠️ Build           | patch  | ![#AA5C3F](https://via.placeholder.com/15/AA5C3F?text=+) | Type: Build       | build:           | プロジェクトのビルドに関する変更     |
| 📦 Dependency      | patch  | ![#7B4334](https://via.placeholder.com/15/7B4334?text=+) | Type: Dependency  | chore(deps):     | 外部依存モジュールに関する変更       |
| 📦 Dependency      | patch  | ![#7B4334](https://via.placeholder.com/15/7B4334?text=+) | Type: Dependency  | chore(deps-dev): | 開発用外部依存モジュールに関する変更 |
| -                   | -      | ![#624498](https://via.placeholder.com/15/624498?text=+) | Type: Release     | -                | リリース判定プルリクエスト           |
| -                   | -      | ![#ECE038](https://via.placeholder.com/15/ECE038?text=+) | Type: Canary      | -                | カナリア・リリース                   |


## まとめ
GitHub Actions でプルリクエストに付くコミットメッセージを取得できます。
これによりコミットメッセージの文字列からプルリクエストのラベルを付けることができます。

lerna-changelog で CHANGELOG.md を自動生成している環境ではプルリクエストのラベルから CHANGELOG.md を作っているので、コミットメッセージからラベルを自動で付与できることは効率的です。そして Conventional Commits などとの開発との組み合わせも良好です。

ラベルの自動付与を入れることでプルリクエストのラベル必須化チェックの導入もしやすくなり、lerna-changelog の CHANGELOG.md 生成をさらに活かせるようになります。

柔軟なスクリプトを組めるのも GitHub Actions のいいところですね。

### 関連する記事
- [サービスを Ship.js でプルリク･ベースにリリースする](https://zenn.dev/lulzneko/articles/using-shipjs-to-make-service-releases-pr-based)
- [lerna-changelog でプルリクからリリースノート生成する](https://zenn.dev/lulzneko/articles/generate-release-notes-by-pr-with-lerna-changelog)
- [GitHub Actions でコミットログからプルリクにラベル貼り](https://zenn.dev/lulzneko/articles/labeling-pr-from-commitlog-with-githubactions) (本記事)


## 参考サイト
- [lerna-changelog](https://github.com/lerna/lerna-changelog)
- [Ship.js](https://community.algolia.com/shipjs/)
