---
title: "SvelteKit と sk-auth でソーシャル認証を簡単に導入する"
emoji: "🗝️"
type: "tech"
topics: [ sveltekit, svelte, oauth, auth, nodejs ]
published: true
announce:
  記事を書きました。
  SvelteKit なサイトに Google などのソーシャル認証を導入する方法を紹介。

  sk-auth という SvelteKit 用の認証ライブラリーを使うことでソーシャル認証を手軽に導入することができます。

  ⇒ SvelteKit と sk-auth でソーシャル認証を簡単に導入する
  https://zenn.dev/lulzneko/articles/easy-to-impl-sns-login-with-sveltekit-and-sk-auth
---

## はじめに
この記事は [Svelte Advent Calendar 2021](https://qiita.com/advent-calendar/2021/svelte) の 16日目の記事です。

ちょっとしたサイトから本格的なサイトまで柔軟に対応できて効率的な開発が行える SvelteKit。

最近 Web API をメインとする開発をしているなかで、やはり仮でよいので Web UI が欲しいとのことで SvelteKit で実装。検証用にも手軽で高速に実装できて助かります。

そんな中、認証が必要となり組み込むことにしました。
SvelteKit 用の認証ライブラリ sk-auth を使うことで、こちらもまた手軽に高速にソーシャル・ログインなどを実装できます。実際の開発としては sk-auth で SAML 認証をしているのですが、いきなり SAML 認証での例示では sk-auth の素晴らしさを十分にお伝えできないかと思い、まずはソーシャル・ログイン系として Google 認証での例を紹介したいと思います。

sk-auth を使うことで SvelteKit のサイトに手軽に認証機能を導入できます。

### 解決したい課題
- ソーシャル・ログインを使いたいが実装が分からない
- ソーシャル・ログインの導入に手間がかかっている
- 複数のソーシャル・ログインに対応すると複雑化する

### 本記事の前提
- Google Cloud Platform の認証情報の設定は完了しているものとします
- リダイレクト URI は `http://localhost:3000/api/auth/callback/google` とします

### 動作環境
本記事は以下の環境で動作を確認しています。
- Node.js 16
- SvelteKit 1.0.0-next.201
- sk-auth 0.3.7
- Ubuntu 18.04 LTS (WSL1) / GitHub Actions Ubuntu


## SvelteKit とは
"素晴らしい開発エクスペリエンス" を備えた、あらゆる規模のウェブ・アプリケーションを構築するためのフレームワークです。
https://kit.svelte.dev/

以下の機能を謳っています。
- ベース・フレームワークの Svelte パワー
- SSR と SPA、両方の世界の最高傑作
- 迅速な構築と飛躍的な成長

まずベース・フレームワークとしての Svelte が強力です。
ボイラープレートいわゆるテンプレなしに HTML/CSS/JavaScript で開発が行え、JavaScript そのものがリアクティブにできるので変数へ代入をそのままにビューを更新できます。それでいて仮想 DOM を使用せずに小さな Vanilla JS としてコンパイルするという特徴があります。
ぜひ [チュートリアル](https://svelte.jp/tutorial/basics) を試してください。Svelte の開発のしやすさを実感できるでしょう。

そして SSRと SPA が両立する世界観です。
SSR のメリットである SEO、プログレッシブ・エンハンスメント、初期ロード・エクスペリエンスが活かされ、SPA のメリットの高速なナビゲーションとアプリのような感覚が共存します。

加えて、高度なルーティング、サーバーサイドレンダリング、コード分割、オフラインサポートなどが上げられています(オフラインサポートはこれから？)。高度なルーティングは本記事で登場する sk-auth の活用例がおもしろいかもしれません。


## sk-auth とは
SvelteKit 用の認証ライブラリです。
https://github.com/Dan6erbond/sk-auth

いくつかのソーシャル・ログインの実装も提供されているため、対応しているものは簡単に導入できます。バージョン 0.3.7 では以下のプロバイダーの実装が提供されています。
- Facebook
- Google
- Reddit
- Twitch
- Twitter

OAuth2 のベース実装もあり設定ベースで導入できます。また独自に [プロバイダーの実装](https://github.com/Dan6erbond/sk-auth#adding-more-providers) も可能です。もちろん、複数のプロバイダー対応も柔軟で手軽に行える設計・実装となっています。


## SvelteKit/sk-auth のセットアップ

### スケルトン・プロジェクトの作成
Google 認証を試すためのスケルトン・プロジェクトを作成します。

```shell-session
npm init svelte@next sveltekit-google-auth
```

インタラクティブ・セットアップでは以下の設定にしました。
- Which Svelte app template? › `Skeleton project`
- Use TypeScript? › `Yes`
- Add ESLint for code linting? › `Yes`
- Add Prettier for code formatting? › `No`

つづいて sk-auth のインストールを行います。
```shell-session
cd sveltekit-google-auth
npm i -D sk-auth
```


### トップページの作成
トップ・ページの `src/routes/index.svelte` は認証状態に応じて表示を変えるようにします。
```html:src/routes/index.svelte
<script>
  import { signOut as authSignOut } from 'sk-auth/client';
  import { session } from '$app/stores';

  $: user = $session.user;

  function signIn() {
    location.assign('/api/auth/signin/google?redirect=/');
  }

  function signOut() {
    authSignOut().then(session.set);
  }
</script>


{#if !user}
  <button on:click="{signIn}">サインイン with Google</button>
{:else}
  <h2>Hello {user.name} !!</h2>
  <p><img src={user.picture} alt="{user.name} icon" /></p>
  <button on:click={signOut}>サインアウト</button>
{/if}
```

ボタンは `<a>` で作りたいとか、ちゃんとレイアウトしてとか考えたくなりますが、実装サンプルなので必要事項だけに絞ります。ボタン１個の画面とかさみしいですけどね。

Session (および Cookie、後述) に有効な認証情報がなければ [サインイン with Google] ボタンを表示。認証情報があればユーザー名と画像の表示をする画面を作ります。

[サインイン with Google] ボタンは `api/auth/signin/google` へアクセスします。ここは SPA ナビゲーションではなく HTTP アクセスする必要があるため `location.assign()` しています。


### Google プロバイダーを構成
認証のプロバイダー `src/lib/auth.ts` を作成します。
```typescript:src/lib/auth.ts
import { Providers, SvelteKitAuth } from 'sk-auth';

export const auth = new SvelteKitAuth({
  jwtSecret: import.meta.env.VITE_JWT_SECRET,
  providers: [
    new Providers.GoogleOAuth2Provider({
      clientId: import.meta.env.VITE_GOOGLE_OAUTH_CLIENT_ID,
      clientSecret: import.meta.env.VITE_GOOGLE_OAUTH_CLIENT_SECRET,
      profile(profile) {
        return { ...profile, provider: 'google' };
      }
    })
  ]
});
```

HTTP リクエストをハンドルしてくれる `SvelteKitAuth` のインスタンスを作ります。
`providers` へ処理するプロバイダーを追加します。今回は Google 認証 `GoogleOAuth2Provider` のインスタンスです。providers が配列になっている通り複数のプロバイダーが追加できます。

`import.meta.env.VITE_XXXX` は環境変数 `VITE_XXXX` から値を受け取るコードです。SvelteKit 1.0.0-next.201 現在では環境変数は `VITE_` プレフィックスが必要です。
※ [How do I use environment variables? - FAQ • SvelteKit](https://kit.svelte.dev/faq#env-vars)

:::message alert
この３つの環境変数をクライアント(画面)側のコードで参照しないでください
参照した場合クライアント側のコードにバンドルされ値が見えてしまいます
:::


### .env ファイルの作成
環境変数を渡すための .env ファイルを作ります。

```shell:.env
# JWT secret used to sign and verify the cookies (256-bits or more)
VITE_JWT_SECRET=[your-jwt-secret-over-256-bit]

# Client ID for Google OAuth
VITE_GOOGLE_OAUTH_CLIENT_ID=[your-google-oauth-client-id]

# Client Secret for Google OAuth
VITE_GOOGLE_OAUTH_CLIENT_SECRET=[your-google-oauth-client-secret]
```

`VITE_JWT_SECRET_KEY` は、Cookie へ保存するユーザー情報の署名と検証に使うシークレットキーです。任意の文字列で構いませんが、実際の運用時には適切な長さと複雑さの文字列を設定し安全に管理してください。少なくとも 256bit 以上、512bit ぐらいの長さがあるとよいでしょう。
※ [HMAC with SHA-2 Functions - rfc7518](https://datatracker.ietf.org/doc/html/rfc7518#section-3.2)

`VITE_GOOGLE_OAUTH_CLIENT_ID` と `VITE_GOOGLE_OAUTH_CLIENT_SECRET` は Google認証を設定した際に取得したクライアント ID とクライアント・シークレットを設定します。

:::message alert
.env ファイルを GitHub へコミットしないでください
.gitignore へ追加するなどして対策をしてください
:::


### 環境変数の型定義の追加
TypeScript の場合は `import.meta.env.XXXX` で型エラーが発生するため定義を追加します。

```typescript:src/global.d.ts
/// <reference types="@sveltejs/kit" />

interface ImportMetaEnv {
  VITE_JWT_SECRET_KEY: string;
  VITE_GOOGLE_OAUTH_CLIENT_ID: string;
  VITE_GOOGLE_OAUTH_CLIENT_SECRET: string;
}
```

余談ですが `ImportMetaEnv` が interface 定義のため、アプリ側にて同じインターフェイス名で追加するとマージされます。TypeScript の type と interface の違いを感じるところですね。


### 認証ハンドラーのエンドポイント作成
認証ハンドラー `SvelteKitAuth` インスタンスを公開するためのエンドポイントを作成します。

`src/routes/api/auth/[...idp].ts` ファイルを追加します。
変なファイル名ですが [Rest parameters](https://kit.svelte.dev/docs#routing-advanced) で、動的なパラメーターとしてルーティングされます。

```typescript:src/routes/api/auth/[...idp].ts
import { auth } from '$lib/auth';
export const { get, post } = auth;
```

HTTP GET と HTTP POST を `$lib/auth` = `SvelteKitAuth` のインスタンスで受けます。あとはライブラリーが、よろしく処理をしてくれます。(と、言うと身も蓋もないのですが。。。)

処理を具体的にみると `/api/auth/[処理]/[プロバイダー]` ような URL を処理しています。
ここでの `/[処理]/[プロバイダー]` は、ログインボタンで指定した `/signin/google` であり、コールバック URI で指定した `/callback/google` です。
- サインイン: `location.assign("/api/auth/signin/google?redirect=/home");`
- リダイレクト: `http://localhost:3000/api/auth/callback/google`

ここでの `[処理]` は、SvelteKitAuth で  `signin` または `callback` と決められてます。
`[プロバイダー]` は、SvelteKitAuth の providers に渡したプロバイダーの ID です。今回は `GoogleOAuth2Provider` を使い内部的に `id = google` が設定されているため `google` です。
(実装個所をリンクしたいのだけどタグが打たれてないから張れない。リリース時のタグ、大事)

SvelteKit の柔軟なルーティングと、これらの組み合わせで最小限の実装で複数のプロバイダーをサポートした認証ハンドラーが実現できています。正に SvelteKit が特徴として謳っている高度なルーティングを活かした実装と言えるのではないでしょうか。


### セッション・ストアの設定
認証したユーザー情報をセッションで扱えるように `src/hooks.js` ファイルを追加します。

```typescript:src/hooks.ts
import { auth } from '$lib/auth';
export const { getSession } = auth;
```


## 動作確認
SvelteKit のローカルサーバーを起動します。

```shell-session
npm run dev -- --open
```

ブラウザが表示されローカルサーバーへ接続されます。
[サインイン with Google] ボタンを押下し Google 認証をします。

認証が成功すると `/home` へリダイレクトされ、Google のユーザー情報が表示されます。

また、認証が成功すると Cookie にユーザー情報の JWT が保存されます。[サインアウト] ボタンを押さずにブラウザを閉じて、再度 `http://localhost:3000` へアクセスすると認証済みの状態となります。これは Cookie に保存されたユーザー情報の JWT 正しく検証され有効期限内(デフォルト30日)だった場合に、認証済みとして処理されるためです。
※ この JWT の署名と検証に `VITE_JWT_SECRET_KEY` を使うため、適切な値を管理が必要です


## まとめ
SvelteKit を使うこと簡単で高速に高品質のウェブサイトを作ることができます。
そして sk-auth を組み合わせることで SvelteKit のサイトに手軽に認証機能を追加できます。

とくにソーシャル・ログイン系を導入する場合は複数のプロバイダー対応する必要がありますが、sk-auth では SvelteKit の高度なルーティングを活用して最小限の実装で実現できます。

またプロバイダーの追加は柔軟に行えるため、本来実装していた SAML 認証も同じ実装で簡単に追加できました。プロバイダー実装だけなのですが、いつかの機会に紹介します。


## 参考サイト
[@robertohuertasm](https://twitter.com/robertohuertasm) さんの記事を参考にさせていただきました。ありがとうございます！
https://robertohuertas.com/2021/07/25/sveltekit-cognito-authentication/

- [Svelte • サイバネティクスで強化されたWebアプリ](https://svelte.jp/)
- [SvelteKit • The fastest way to build Svelte apps](https://kit.svelte.dev/)
- [Dan6erbond/sk-auth](https://github.com/Dan6erbond/sk-auth)
- [Dan6erbond/Cheeri-No](https://github.com/Dan6erbond/Cheeri-No)
- [Cognito Authentication for your SvelteKit app | Roberto Huertas](https://robertohuertas.com/2021/07/25/sveltekit-cognito-authentication/)
