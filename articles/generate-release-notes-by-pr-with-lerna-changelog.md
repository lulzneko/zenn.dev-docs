---
title: "lerna-changelog でプルリクからリリースノート生成する"
emoji: "📘"
type: "tech"
topics: [ release, cicd, githubactions, github, nodejs ]
published: true
announce:
  記事を書きました。
  Ship.js を使ったサービス系リリースに、プルリクを元にリリースノートを生成する機能を追加。

  プルリク・ベースなのでラベルによる分類と柔軟な項目の管理と整理ができます。

  ⇒ lerna-changelog でプルリクからリリースノート生成する
  https://zenn.dev/lulzneko/articles/generate-release-notes-by-pr-with-lerna-changelog
---

## はじめに
いくつかのサービス開発に携わってきた中、リリース管理についてサービスごとにまちまちで、開発者フレンドリーではない点を改善したく悩んできました。

その中 Ship.js を使うことでリリース判定をプルリクエスト・ベースにし、判定の内容と証跡をソースコードとともに GitHub へ残せるようにしました。
https://zenn.dev/lulzneko/articles/using-shipjs-to-make-service-releases-pr-based

運用については順調ですがリリースノートの生成について Ship.js デフォルトの [Conventional Commits](https://www.conventionalcommits.org/) は、サービスのリリースに上手くはまらなかったためカスタマイズしました。

プルリクエストを主体とした、サービスのリリースノート生成を紹介します。

### 解決したい課題
- リリースノートの作成に手作業が多く発生している
- 項目の粒度が細かすぎる、あるいは適切な粒度作成が難しい
- 自動化したがリリース項目と分類を後から変更できない

### 本記事の前提
- [サービスのリリースを Ship.js でプルリクエスト・ベースにする](https://zenn.dev/lulzneko/articles/using-shipjs-to-make-service-releases-pr-based) の記事の環境とします
- ソースコードの変更をプルリクエスト・ベースで行っているものとします
- リリースノート = 変更ログ(CHANGELOG.md) とします

### 動作環境
本記事は以下の環境で動作を確認しています。
- Node.js 16
- lerna-changelog 2.1.0
- Ubuntu 18.04 LTS (WSL1) / GitHub Actions Ubuntu


## lerna-changelog とは
複数パッケージを持つ JavaScript プロジェクトの管理(いわゆるモノレポ)ツールの [Lerna](https://lerna.js.org/) が同じく OSS で開発しているプルリクエスト・ベースの変更ログ生成ツールです。
https://github.com/lerna/lerna-changelog

主に以下の機能を提供しています。
- GitHub リポジトリのタグが付いた最新のコミット以降のプルリクエストを収集する
- 特定のラベルが付いたプルリクエストを変更ログとして出力する
- レンジを指定した同様のプルリクエスト収集と変更ログの出力

プルリクエストに設定したラベルをベースとして変更ログを出力してくれます。
デフォルトでは `breaking`、`enhancement`、`bug`、`documentation`、`internal` に対応します。


## プルリクエストを活用するメリット
リリースノートはユーザーや関連システムの開発者などへ伝える情報がメインとなります。
主に公開されている機能に関することで、新しい機能の追加や、既存機能の変更などです。逆にサービスの内部的なことやコミット毎の細かい変更情報といったものは不要となります。

Ship.js のデフォルトである Conventional Commits はコミット・メッセージ単位となり粒度が細かすぎることと内容がコードの変更にフォーカスするので公開機能レベルにはなりません。
(むしろ公開機能レベルではコミット粒度が大きすぎるため、それはそれで困ります)

プルリクエストをリリースノートの項目レベルにし分類や記載の有無にラベルを使うことで、リリースノートに記載する内容を適切に制御できます。そしてプルリクエストの記載内容(ソースコードではなく)は「後から修正できる」ことも大きなメリットとなります。

開発時は開発を進めることが中心でConventional Commits でも、型やスコープのミスやコミット粒度の間違えは発生します。開発・進捗することが目的なので仕方のない部分もあります。

いっぽうでプルリクエストは「レビュー」です。プルリクエストの適切性やコードの正しさなどをチェックであり足を止めて見直すことです。必然的にプルリクエストの目的にあっているか、コミット単位や範囲は適切かなどを見定めます。タイトルやラベルなども修正できます。マージ後でも変更可能であり最終レビューの場であるリリース判定でも見直して修正できるのです。

これはサービスのリリースという公開機能を中心とした抽象度の高い項目を扱うことに有利で、コミット・メッセージの習慣を壊すことなく Conventional Commits とも共存できます。

サービスのリリースノートにプルリクエストを使うのはメリットとが大きいといえるでしょう。


## lerna-changelog のセットアップ

### インストール
npm モジュールを追加します。

```shell-session
npm i -D lerna-changelog
```


### 基本的な設定
package.json にリリースノートへ生成する GitHub プルリクエストのラベルを定義します。
また repository 要素も必要なため設定されてない場合は追加します。

changelog/labels に、`"ラベル名": "リリースノートへ記載する見出し"` として定義します。
ラベル名は GitHub プルリクエストのラベルと一致する必要があります。

今回は新しくラベルを作成し、同じく GitHub のリポジトリへも [ラベルを作成](https://docs.github.com/ja/issues/using-labels-and-milestones-to-track-work/managing-labels) しました。
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
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/[your-org]]/[your-repository].git"
  }
}
```

※ 設定を行いリリース前のプルリクエストに対応するラベルを付けたら lerna-changelog は実行可能です。後からプルリクエストにラベルを追加できるメリットが早くも享受できますね。
- [GitHub の PAT(Personal access token) を作成](https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) する (Ship.js の利用で作ったもので OK)
- `GITHUB_AUTH=[your-pat] npx lerna-changelog` (最後のバージョン・タグ以降の最新)
- `GITHUB_AUTH=[your-pat] npx lerna-changelog --from=v1.0.0 --to=v2.0.0` (特定のタグ間)


### Ship.js との連携
[サービスのリリースを Ship.js でプルリクエスト・ベースにする](https://zenn.dev/lulzneko/articles/using-shipjs-to-make-service-releases-pr-based) の記事の環境へ lerna-changelog を組み込んでいきます。ship.config.js に以下の内容を追加します。

```javascript:ship.config.js
import { spawnSync } from 'child_process';
import { readFile, readFileSync, writeFileSync } from 'fs';
import { exit } from 'process';
import { inc } from 'semver';

const command = './node_modules/.bin/lerna-changelog';
const file = 'CHANGELOG.md';

module.exports = {
  updateChangelog: false,

  getNextVersion: ({ currentVersion }) => {
    const spawn = spawnSync(command);
    if (spawn.status === 0) {
      const stdout = spawn.stdout.toString();
      if (stdout.includes('#### 💥 Breaking Change')) { return inc(currentVersion, 'major'); }
      if (stdout.includes('#### 🎉 New Feature') || stdout.includes('#### 🚀 Enhancement')) { return inc(currentVersion, 'minor'); }
      if (stdout.includes('####')) { return inc(currentVersion, 'patch'); }
      console.log('No changes');
      return exit(spawn.status);
    }
    if (spawn.stdout) { console.log(spawn.stdout.toString()); }
    if (spawn.stderr) { console.error(spawn.stderr.toString()); }
    return exit(spawn.status);
  },

  beforeCommitChanges: ({ nextVersion }) => new Promise((resolve) => {
    const spawn = spawnSync(command, [ '--next-version', `v${nextVersion}` ]);
    if (spawn.status === 0) {
      readFile(file, 'utf8', (_err, changelog) => {
        writeFileSync(file, `${spawn.stdout.toString().trim()}\n${changelog ? `\n\n${changelog}` : ''}`);
        resolve();
      });
      return;
    }
    if (spawn.stdout) { console.log(spawn.stdout.toString()); }
    if (spawn.stderr) { console.error(spawn.stderr.toString()); }
    exit(spawn.status);
  }),

  const releases = {
    extractChangelog: () => {
      const changelog = readFileSync(file, 'utf-8').toString();
      return changelog.split('\n\n\n')[0].replace(/## .+\n\n/ug, '').replace('/####/g', '###');
    }
  }
};
```

:::details ship.config.js 全文
```javascript:ship.config.js
import { spawnSync } from 'child_process';
import { readFile, readFileSync, writeFileSync } from 'fs';
import { exit } from 'process';
import { inc } from 'semver';

const command = './node_modules/.bin/lerna-changelog';
const file = 'CHANGELOG.md';

module.exports = {
  updateChangelog: false,
  draftPullRequest: true,
  pullRequestTeamReviewers: [ 'product-owners' ],

  buildCommand: undefined,
  installCommand: () => '# do nothing',
  publishCommand: () => '# git merge production master in afterPublish hook',

  getNextVersion: ({ currentVersion }) => {
    const spawn = spawnSync(command);
    if (spawn.status === 0) {
      const stdout = spawn.stdout.toString();
      if (stdout.includes('#### 💥 Breaking Change')) { return inc(currentVersion, 'major'); }
      if (stdout.includes('#### 🎉 New Feature') || stdout.includes('#### 🚀 Enhancement')) { return inc(currentVersion, 'minor'); }
      if (stdout.includes('####')) { return inc(currentVersion, 'patch'); }
      console.log('No changes');
      return exit(spawn.status);
    }
    if (spawn.stdout) { console.log(spawn.stdout.toString()); }
    if (spawn.stderr) { console.error(spawn.stderr.toString()); }
    return exit(spawn.status);
  },

  beforeCommitChanges: ({ nextVersion }) => new Promise((resolve) => {
    const spawn = spawnSync(command, [ '--next-version', `v${nextVersion}` ]);
    if (spawn.status === 0) {
      readFile(file, 'utf8', (_err, changelog) => {
        writeFileSync(file, `${spawn.stdout.toString().trim()}\n${changelog ? `\n\n${changelog}` : ''}`);
        resolve();
      });
      return;
    }
    if (spawn.stdout) { console.log(spawn.stdout.toString()); }
    if (spawn.stderr) { console.error(spawn.stderr.toString()); }
    exit(spawn.status);
  }),

  afterPublish: ({ exec }) => {
    exec('git config --global user.email "github-actions[bot]@users.noreply.github.com"');
    exec('git config --global user.name "github-actions[bot]"');
    exec('git checkout production');
    exec('git merge master --ff');
    exec('git push origin production');
  },

  releases: {
    extractChangelog: () => {
      const changelog = readFileSync(file, 'utf-8').toString();
      return changelog.split('\n\n\n')[0].replace(/## .+\n\n/ug, '').replace('/####/g', '###');
    }
  }
};
```
:::

以下のように設定しています。

`updateChangelog` で Ship.js による変更ログの生成を無効化します。

`getNextVersion` は、次のリリース・バージョンを決めます。
lerna-changelog を実行し変更ログを生成します。その変更ログの文字列から増加させるバージョンを決めます。package.json の changelog に設定した `リリースノートへ記載する見出し` と合わせる必要があることに注意してください。今回のルールは以下です。
- `#### 💥 Breaking Change` が含まれていたらメジャー・バージョンアップ
- `#### 🎉 New Feature` か `#### 🚀 Enhancement` の場合はマイナー・バージョンアップ
- `####` (上記以外の見出し) の場合はパッチ・バージョンアップ
- 上記以外の場合は変更なしとして終了

`beforeCommitChanges` は、リリースの変更コミット前の処理で変更ログのファイルを生成します。

`releases` は、GitHub のリリース・タブにアップロードする変更ログを生成します。
今回のリリース分だけを切り出しています。


### Dry Run スクリプトの追加
生成される変更ログを確認できるように Dry Run のスクリプトを追加します。
プルリクエストのタイトルやラベルは後から変更できますがリリース・プルリクエストを作ってからは手作業の修正になるため、あらかじめ確認し修正しておくとよいでしょう。

```json:package.json
{
  "scripts": {
    "release": "shipjs prepare",
    "release:dry": "shipjs prepare --dry-run && node -r dotenv/config ./node_modules/.bin/lerna-changelog"
  }
}
```


### リリース実行環境の準備
Ship.js の設定で作成済みの `.env` ファイルに設定を追加します。

```shell:.env
# A GitHub personal access tokens to grant shipjs access to GitHub repository
GITHUB_TOKEN=[your-pat]

# A GitHub personal access tokens to grant lerna-changelog access to GitHub repository
GITHUB_AUTH=[your-pat]
```

:::message alert
.env ファイルを GitHub へコミットしないでください
.gitignore へ追加するなどして対策をしてください
:::


## リリース
Ship.js のコマンドで、lerna-changelog がプルリクエストから収集した変更ログを生成します。
```shell-session
npm run release
```


## まとめ
lerna-changelog を使うことで、プルリクエスト・ベースの変更ログを生成できます。

プルリクエストとラベル活用し公開機能や既存機能の変更といったリリース・ノートに必要な単位を収集することでサービスのリリース・ノートに必要な情報を自動で生成できます。

プルリクエストを使うことで、タイトルやラベルは後からでも修正できるのでリリース時に確認して整理することもできます。この柔軟さがプルリクエストを使うことのメリットでしょう。

いっぽうで Conventional Commits でコミット・メッセージを適切に運用できており、そのままプルリクエストのラベルと結びつくようなケースではラベルを付けるのが手間になってしまうこともあります。このようなケースをサポートするための GitHub Actions を組んでいるので、またご紹介できればと思います。


### 関連する記事
- [サービスを Ship.js でプルリク･ベースにリリースする](https://zenn.dev/lulzneko/articles/using-shipjs-to-make-service-releases-pr-based)
- [lerna-changelog でプルリクからリリースノート生成する](https://zenn.dev/lulzneko/articles/generate-release-notes-by-pr-with-lerna-changelog) (本記事)
- [GitHub Actions でコミットログからプルリクにラベル貼り](https://zenn.dev/lulzneko/articles/labeling-pr-from-commitlog-with-githubactions)


## 参考サイト
[@kazu_pon](https://twitter.com/kazu_pon) さんの記事を参考にさせていただきました。ありがとうございます！
https://qiita.com/kazupon/items/0038529541c1e59e9124

- [lerna-changelog](https://github.com/lerna/lerna-changelog)
- [Ship.js](https://community.algolia.com/shipjs/)
- [Ship.jsでリリースフローを改善する - Qiita](https://qiita.com/kazupon/items/0038529541c1e59e9124)
