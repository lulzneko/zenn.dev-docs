---
title: "サービスを Ship.js でプルリク･ベースにリリースする"
emoji: "🛳"
type: "tech"
topics: [ release, cicd, githubactions, github, nodejs ]
published: true
announce:
  記事を書きました。
  リリース管理を GitHub に集約する方法を紹介。

  Ship.js という OSS のリリース管理ツールをサービス系リリースに応用してシステマチックでコードと判定・承認を GitHub で一元管理する方法です

  ⇒ サービスを Ship.js でプルリク･ベースにリリースする
  https://zenn.dev/lulzneko/articles/using-shipjs-to-make-service-releases-pr-based
---

## はじめに
いくつかのサービス開発に携わってきた中、リリース管理についてサービスごとにまちまちで、開発者フレンドリーではない点を改善したく悩んできました。

一番使ってきた方法は Wiki などのツールにリリース内容や判定情報を記述し判定会を開催する方法です。この Wiki が意外と曲者で情報を集める必要があり、ソースコードやプルリクエストなどのリンクをコピペする手間がありました。これは仕方がないとしても、ツール変更による移転作業や情報のロスト、何よりも証跡を適切な形で保持しておけない事などの問題がありました。

そうした中 [@kazu_pon](https://twitter.com/kazu_pon) さんの [Ship.js でリリースフローを改善する - Qiita](https://qiita.com/kazupon/items/0038529541c1e59e9124) から着想を得て、サービス系のリリース管理を GitHub のプルリクエストで行う方法を作り運用を開始しました。

現在のところ順調にリリース・プロセスが運用できているので、その方法を紹介します。
さまざまな機能を組み合わせており、全体像は１回で書ききれないのでパートパートでの紹介となりますが、まずは最初の一歩としてリリース判定と承認を GitHub で行う方法を紹介します。

### 解決したい課題
- サービスのリリース手順に手作業が多く発生している
- リリース判定がソースコードと分離して管理されている
- リリース判定の証跡が曖昧かつ承認と実行が乖離している

### 本記事の前提
- ソースコードは GitHub で管理されているものとします
- デプロイなどの作業は CI/CD で自動化されているものとします
- バージョンの採番と更新、変更ログ作成、Git のタグ打ちなどの作業の削減を目標とします

### 動作環境
本記事は以下の環境で動作を確認しています。
- Node.js 16
- Ship.js 0.24.0
- Ubuntu 18.04 LTS (WSL1) / GitHub Actions Ubuntu


## Ship.js とは
ウェブ検索の SaaS で有名な [Algolia](https://www.algolia.com/) が OSS で開発しているリリース管理のためのツールです。
https://github.com/algolia/shipjs

大きく、以下の３つの機能を謳っています。

1. 自動化
リリース作業を最小化し、ミスを減らすことができる

2. 非同期
ローカルマシンでリリースを行わず、リリースを非同期化することで仕事を続けられる

3. 共同作業
次のリリースをプルリクエストのレビューとして同僚と一緒に検討できる

つまり、リリースに付帯するバージョンの採番や package.json の更新、タグ打ちといった定型的な作業を自動化しミスを減らしてくれます。これを GitHub Actions などの CI/CD を通じで実行するので、ローカルマシンのリソースを取られることなく自身の作業が続けられます。リリースの処理待ちで他のことできないとか避けたいですね。

なにより特徴的なのが、リリースのレビューをプルリクエストで行えることです。
Ship.js はリリース処理を行う前にプルリクエストを作るので、関係者でリリースバージョンや、リリース内容の確認、変更ログ/リリースノート(CHANGELOG.md) の確認と修正が行えます。

これらは npm モジュールのリリースを前提としてはいますが、サービスとしてもリリース判定をプルリクエストで行うと考えることで十分に適用できます。

むしろソースコードと同じ場所にリリース判定と承認の証跡を残せることのメリットの方が大きいのではないでしょうか。


## Ship.js を使ったリリース・フロー
以下の手順でリリースを行います。
1. 開発者がリリースをしたいタイミングで `npm run release` を実行しリリース判定のドラフト・プルリクエストを作成します
2. プルリクエストにリリース判定用のコメントを追加し [Ready for review] します
3. リリース判定会を開催しリリース内容を評価します
4. 承認が得られた場合、承認者がリリース判定プルリクエストを [Squash and merge] します


## Ship.js のセットアップ

### インストール
Ship.js をインストールし実行すると、インタラクティブ・セットアップが起動します。

```shell-session
npm i -D shipjs
npx shipjs setup
```

デフォルトのブランチ、CI、リリースをスケジュール(cron)するか、をプロジェクトに合わせて設定することで完了です。

今回は以下のように実行しました。

```shell-session
$ npx shipjs setup
? What is your base branch?
  This is also called "Default branch".
  You usually merge pull-requests into this branch. main
? Which CI configure? GitHub Actions
? Add manual prepare with issue comment?
  You can create `@shipjs prepare` comment on any issue and then github actions run `shipjs prepare` Yes
? Schedule your release? No
› Installing Ship.js
› Adding scripts to package.json
› Creating ship.config.js
› Creating GitHub Actions configuration

🎉  FINISHED
```

package.json の scripts に `release` が追加されます。
また CI を選択した場合は、その CI の設定も追加されます。

※ `.github/workflows/shipjs-manual-prepare.yml` は、GitHub Issues に `@shipjs prepare` とコメントすると、リリース準備を実行する GitHub Actions です。通常は、後述する開発者のコマンドラインから実行しますが、GitHub からも実行した場合に使います。


### 基本的な設定
Ship.js の設定は、プロジェクトのルートディレクトリ直下の `ship.config.js` で行います。
デフォルトでは npm モジュールの公開として動作しますので、これをサービスのリリース用にカスタマイズします。

サービスのリリース用のポイントは以下です。
- `publishCommand` を無効化する (あるいはリリース用コマンドを設定する)
- `afterPublish` でリリース用のコマンド群を実行する

今回は以下のように設定しました。

```javascript:ship.config.js
import { readFileSync } from 'fs';

module.exports = {
  draftPullRequest: true,
  pullRequestTeamReviewers: [ 'product-owners' ],

  buildCommand: undefined,
  installCommand: () => '# do nothing',
  publishCommand: () => '# git merge production master in afterPublish hook',

  afterPublish: ({ exec }) => {
    exec('git config --global user.email "[your-email]"');
    exec('git config --global user.name "[your-name]"');
    exec('git checkout production');
    exec('git merge master --ff');
    exec('git push origin production');
  }
};
```

以下のように設定しています。

リリース用プルリクエストをドラフトで作成し、`product-owners` をチームレビュアーとしてアサインしています。(レビュアーは開発体制により変わると思いますが、リリースのレビュアー = 承認者なのでプロダクト・オーナーとしています)

`buildCommand`、`installCommand` を無効化しています。これは CI で別に行うためです。

`publishCommand` も無効化します。ここはデフォルトで `npm publish` が実行されますが、サービスリリースでは不要です。
もし１つのコマンドでリリースが行える場合は、そのコマンドを記述します。
今回は１コマンドでできないので無効化し、`afterPublish` で実行するため無効化しました。

`afterPublish` は、リリース処理(`publishCommand`) 実行後の処理を記述します。
今回は `publishCommand` を無効化しリリース作業をとして、ここにプロジェクトごとのリリースコマンドを書いていきます。

ブランチごとの CI/CD が別で組まれている場合は、それをキックすることもできます。
今回は production の CI/CD を起動するため、マージ処理を行うようにしました。

また `publishCommand` を無効化して、`afterPublish` なしの場合はタグを打つだけの動きになります。
タグを打つだけの運用でも使えるので便利です。

※ Ship.js は、ほとんどの処理ステップをカスタマイズでき柔軟なリリース・フローが作れます
https://community.algolia.com/shipjs/reference/all-config.html


### リリース実行環境の準備
リリース判定プルリクエストを作成する開発者は [GitHub の PAT(Personal access token) を作成](https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)し、`.env` ファイルに設定します。
PAT の権限は `repo: Full control of private repositories` が必要です。

```shell:.env
# A GitHub personal access tokens to grant shipjs access to GitHub repository
GITHUB_TOKEN=[your-pat]
```

:::message alert
.env ファイルを GitHub へコミットしないでください
.gitignore へ追加するなどして対策をしてください
:::


## Ship.js でリリース

### プルリクエストを起票
リリースを行いたいタイミングで以下のコマンドを実行します。

```shell-session
npm run release
```

今回のリリース範囲のコミットのリストと、次のバージョン番号の候補が提示されるので確認して進めます。コマンド実行後、自動的にブラウザが起動され作成されたリリース判定用のプルリクエストが表示されます。
判定に必要な内容をコメントし [Ready for review] します。

※ v1.0.0 のように `v` から始まるバージョンタグがない場合、リリース開始のコミットを聞かれます。前回のリリース後のコミットを指定するか、あらかじめ前回のリリースに `vX.X.X` のタグを追加しておきます。

※ Ship.js で自動生成する changelog は、Conventional Commits に従ったコミット・メッセージから生成されます。(ここを [lerna-changelog](https://github.com/lerna/lerna-changelog) を使うように変更していますが、また別のお話)

https://www.conventionalcommits.org/ja/v1.0.0/


### リリース判定とマージ(= 承認)
内容がリリース判定ということ以外に、基本的に通常のプルリクエストのレビューに変わりありません。必要なことをコメントでやり取りしたり、必要に応じて口頭での確認などをします。

実際の運用としてはサービスのリリースなので、ユーザーや関係者へ影響がある場合は判定会議を開催することが多いです。それ以外のリファクタリングなどの内部的な変更にとどまる場合は書面レビューとしてプルリクエスト内のやり取りで進めています。

無事、承認となるとプルリクエストをマージしてクローズします。
このときに **[Squash and merge]** を選択します。Ship.js 固有の対応のため注意が必要です。

あとは `shipjs-trigger.yml` から `npx shipjs trigger` が起動され、`shipjs.config.js` で定義した `publishCommand` と `publishCommand` が実行されます。


## まとめ
Ship.js を使うことで、リリースをシステマチックにできます。
そして、それはサービスのリリースにも応用できます。

これによりバージョンの採番からファイルの更新やタグ打ちなどの提携作業は自動化でき作業を軽減できますし、ミスなどが混入する余地を減らせます。

何よりリリース判定の内容と証跡をソースコードと一緒に管理できるメリットを享受できます。

今回は Ship.js のパートのみ紹介でしたが、実際に使っている環境では以下のようなカスタマイズをも入れています。いずれ機会があったら紹介したいと思います。
- 変更ログの収集を [lerna-changelog](https://github.com/lerna/lerna-changelog) へ変更しプルリクエスト・ベースにする
- Ship.js が作ったリリース判定用のプルリクエストに判定材料の情報を自動コメントする
- master/stable/production の３ブランチ運用へ Ship.js リリース・フローを適用する


## 参考サイト
[@kazu_pon](https://twitter.com/kazu_pon) さんの記事を参考にさせていただきました。ありがとうございます！
https://qiita.com/kazupon/items/0038529541c1e59e9124

- [Ship.js](https://community.algolia.com/shipjs/)
- [Ship.jsでリリースフローを改善する - Qiita](https://qiita.com/kazupon/items/0038529541c1e59e9124)
