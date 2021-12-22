---
title: "SvelteKit の SSR を AWS 環境でサーバーレスに動かす"
emoji: "⚡"
type: "tech"
topics: [ sveltekit, svelte, serverless, aws, lambda ]
published: true
announce:
  記事を書きました。
  SvelteKit + sk-auth(Google OAuth2) の SSR を AWS でサーバーレスに動かす方法を紹介。

  Lambda ベースなのでインスタンスやコンテナーのメンテなしに SSR を運用することができます。

  ⇒ SvelteKit の SSR を AWS 環境でサーバーレスに動かす
  https://zenn.dev/lulzneko/articles/running-sveltekit-ssr-serverlessly-in-aws
---

## はじめに
この記事は [AWS LambdaとServerless Advent Calendar 2021](https://qiita.com/advent-calendar/2021/lambda) の 22日目の記事です。

私がウェブサイトやアプリを作る際、フロントエンドは Jamstack や SSG で静的化します。そうすることで API は FaaS や DBaaS を活用し、フロントエンドは静的サイト・ホスティングが利用できサーバーレスな構成で、そのメリットを享受できるからです。

そのためフロントエンドのフレームワークが SSR と SSG が両方サポートされている場合は当然のように SSG を選びますし、SSR のみの場合は避けるくらいに Jamstack/SSG 派です。

SvelteKit は Adapter とよばれるモジュールを切り替えることで、さまざまなプラットフォーム向けのデプロイセットや Jamstack/SSG などの静的化ができます。

SvelteKit  でも SSG を考えていましたが以下の理由から、はじめて SSR を使うことにしました。
- SvelteKit は SSR + SPA が両立するフレームワークであり、その機能を活かしてみたい
- 認証ライブラリーの sk-auth は SSR が必要
- 認証用 API を別構築はしたくなくフロントエンド側でカバーしたい

SSR をするからといって、サーバーレス構成は崩したくなく、SvelteKit SSR をサーバーレスに動かす構成を作りましたので紹介します。

### 解決したい課題
- SvelteKit で SSR をするがサーバーを構築したくない
- AWS でサーバーレスな SSR 構成が作れない

### 本記事の前提
- [SvelteKit と sk-auth でソーシャル認証を簡単に導入する](https://zenn.dev/lulzneko/articles/easy-to-impl-sns-login-with-sveltekit-and-sk-auth) をベースに開発します
- AWS および Serverless Framework の設定は必要最小限の紹介とします

### 動作環境
本記事は以下の環境で動作を確認しています。

開発環境
- Node.js 16
- SvelteKit 1.0.0-next.201
- sk-auth 0.3.7
- adapter-netlify [6221e6a](https://github.com/sveltejs/kit/tree/6221e6a5758cd1e6cc0f99ccd2ae0e260b97ce3b/packages/adapter-netlify)
- Ubuntu 18.04 LTS (WSL1)

稼働環境
- Amazon CloudFront
- AWS Lambda / Amazon API Gateway (HTTP)
- Amazon S3
- Google Cloud Platform (OAuth 2.0)


## Lambda 用の Adapter を実装
Lambda 用の Adapter がないので自前で実装することにします。

[公式ドキュメント](https://kit.svelte.dev/docs#adapters-writing-custom-adapters)に "The adapter API may change before 1.0." とあるので、本体の変更に加えて、さらに変わる可能性が高いのでしょう。暫定用としてライトな実装を目指します。


### adapter-netlify のソースをコピー
実装にあたり [adapter-netlify](https://github.com/sveltejs/kit/tree/master/packages/adapter-netlify) を流用します。Netlify Functions は、いわば Lambda です。シグネチャーを合わせれば基本的に同じと考えてよいでしょう。また公式ベースにできてよいです。

プロジェクト直下の `adapter-aws-apigw` ディレクトリに対応するファイルをコピーします。

```shell-session
mkdir -p adapter-aws-apigw/files
curl https://raw.githubusercontent.com/sveltejs/kit/6221e6a5758cd1e6cc0f99ccd2ae0e260b97ce3b/packages/adapter-netlify/index.js > adapter-aws-apigw/index.js
curl https://raw.githubusercontent.com/sveltejs/kit/6221e6a5758cd1e6cc0f99ccd2ae0e260b97ce3b/packages/adapter-netlify/files/entry.js > adapter-aws-apigw/files/entry.js
curl https://raw.githubusercontent.com/sveltejs/kit/6221e6a5758cd1e6cc0f99ccd2ae0e260b97ce3b/packages/adapter-netlify/files/shims.js > adapter-aws-apigw/files/shims.js
```


### Adapter の実装を調整
本体である `index.js` から Netlify Functions 専用のコードや、npm モジュールではなくローカルであるためのパスの調整、名前の修正などをします。diff だと分かりにくいので全文載せます。

```javascript:adapter-aws-apigw/index.js
import { writeFileSync } from 'fs';
import { join } from 'path';
import { fileURLToPath } from 'url';
import esbuild from 'esbuild';

/**
 * @typedef {import('esbuild').BuildOptions} BuildOptions
 */

/** @type {import('.')} */
export default function (options) {
  return {
    name: '@sveltejs/adapter-aws-apigw',

    async adapt({ utils }) {
      const publish = 'build';

      utils.log.minor(`Publishing to "${publish}"`);

      utils.rimraf(publish);

      const files = fileURLToPath(new URL('./files', import.meta.url));

      utils.log.minor('Generating serverless function...');
      utils.copy(join(files, 'entry.js'), '.svelte-kit/aws-apigw/entry.js');

      /** @type {BuildOptions} */
      const default_options = {
        entryPoints: ['.svelte-kit/aws-apigw/entry.js'],
        outfile: '.aws-apigw/functions-internal/__render.js',
        bundle: true,
        inject: [join(files, 'shims.js')],
        platform: 'node',
        minify: true
      };

      const build_options =
        options && options.esbuild ? await options.esbuild(default_options) : default_options;

      await esbuild.build(build_options);

      writeFileSync(join('.aws-apigw', 'package.json'), JSON.stringify({ type: 'commonjs' }));

      utils.log.minor('Prerendering static pages...');
      await utils.prerender({
        dest: publish
      });

      utils.log.minor('Copying assets...');
      utils.copy_static_files(publish);
      utils.copy_client_files(publish);
    }
  };
}
```


### エントリーポイントを調整
Lambda 呼び出しのエントリーポイントである `entry.js` の `handler()` 関数を調整します。
関数実行時の引数 `event` が Netlify Functions と Lambda で若干異なるので合わせこみです。同じだったら嬉しかったのですが、さすがにそうもいかなかったのでしょう。

```javascript:adapter-aws-apigw/files/entry.js
export async function handler(event) {
  const { rawPath, requestContext, headers, rawQueryString, body, isBase64Encoded, cookies } = event;

  const query = new URLSearchParams(rawQueryString);

  const encoding = isBase64Encoded ? 'base64' : headers['content-encoding'] || 'utf-8';
  const rawBody = typeof body === 'string' ? Buffer.from(body, encoding) : body;

  headers.cookie = cookies ? cookies.join('; ') : '';

  const rendered = await render({
    method: requestContext.http.method,
    headers,
    path: rawPath,
    query,
    rawBody
  });
  // ...(省略)
```

※ API Gateway のイベントの詳細はこちら
　 [HTTP API の AWS Lambda プロキシ統合の使用 - Amazon API Gateway](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html)


### SvelteKit へ組み込み
設定ファイルの svelte.config.js で、作成した Adapter を組み込みます。
`@sveltejs/adapter-auto` の代わりに追加した `adapter-aws-apigw` をインポートするだけです。

```javascript:svelte.config.js
import adapter from './adapter-aws-apigw/index.js';
// ...(省略)
```

これで `npm run build` を実行すると `.aws-apigw` ディレクトリへ Lambda 用のデプロイ・コードが生成されます。ただし `.aws-apigw` は SSR 部分のみで、静的コンテンツは `build` です。
この２つを上手く AWS へ配置し適切にルーティングする必要があります。


## Serverless で AWS 環境を構築
Lambda 用コードが出力できるようになりましたが、SSR 部分は Lambda へ、静的コンテンツは S3 へ配置しアクセスさせる必要があります。それも同じドメインで。。。

そのためには、さまざまな AWS のサービスを組み合わせる必要があります。そして Lambda コードのデプロイに静的コンテンツのアップロード。これらを一手に担ってくれるのが [Serverless Framework](https://www.serverless.com/framework/docs) です。他にもソリューションがありますが、AWS 内のリソース管理の観点から好んで使っています。ここでは他のツールへポートしやすいように最小限の設定に絞って書きます。

構成のイメージは以下です。CloudFront でリクエストを受付け、静的コンテンツは S3 へ。S3 でコンテンツが無かったら API Gateway/Lambda の SSR へ回します。
また、OAuth2 のフローは SSR 側で URL 生成などの処理や検証などを行います。sk-auth が発行する認証済み JWT の Cookie も SSR で必要になります。
![SvelteKit SSR in AWS](/images/sveltekit-ssr-in-aws.png)


### Serverless のインストール
設定ファイルを TypeScript でチェックしたいので @types/serverless と ts-node も入れます。

```shell-session
npm i -D serverless ts-node @types/serverless  
```

SvelteKit の静的コンテンツを S3 へアップロードする [serverless-s3-deploy](https://github.com/funkybob/serverless-s3-deploy) も追加します。

```shell-session
npm i -D serverless-s3-deploy
```


### Serverless の設定
プロジェクト直下に設定ファイル `serverless-config.ts` を作成します。
設定が長いので全文は下記アコーディオンへ。ここでは抜粋して設定のポイントを説明します。

:::details serverless-config.ts 全文
```typescript:serverless-config.ts
import type { Serverless } from 'serverless/aws';
import { name } from './package.json';

module.exports = ((): Serverless => ({

  service: name,

  provider: {
    name: 'aws',
    runtime: 'nodejs14.x',
  },

  plugins: [
    'serverless-s3-deploy'
  ],

  custom: {
    assets: {
      auto: true,
      targets: [{
        bucket: { Ref: 'SvelteKitStaticContentsBucket' },
        empty: true,
        files: [{
          source: 'build',
          globs: '**'
        }]
      }]
    }
  },

  package: {
    individually: true,
    patterns: [
      '!**',
      '.aws-apigw/**'
    ]
  },

  functions: {
    sveltekit: {
      handler: '.aws-apigw/functions-internal/__render.handler',
      events: [
        { httpApi: { method: '*', path: '*' }}
      ]
    }
  },

  resources: {
    Resources: {
      SvelteKitStaticContentsBucket: {
        Type: 'AWS::S3::Bucket', Properties: {}
      },

      SvelteKitStaticContentsBucketPolicy: {
        Type: 'AWS::S3::BucketPolicy',
        Properties: {
          Bucket: { Ref: 'SvelteKitStaticContentsBucket' },
          PolicyDocument: {
            Statement: [{
              Effect: 'Allow',
              Action: [ 's3:GetObject' ],
              Resource: { 'Fn::Join': [ '/', [{ 'Fn::GetAtt': [ 'SvelteKitStaticContentsBucket', 'Arn' ]}, '*' ]]},
              Principal: {
                CanonicalUser: { 'Fn::GetAtt': [ 'SvelteKitCloudFrontOriginAccessIdentity', 'S3CanonicalUserId' ]}
              }
            }]
          }
        }
      },

      SvelteKitCloudFrontOriginAccessIdentity: {
        Type: 'AWS::CloudFront::CloudFrontOriginAccessIdentity',
        Properties: {
          CloudFrontOriginAccessIdentityConfig: { Comment: name }
        }
      },

      SvelteKitSsrApiGatewayOriginRequestPolicy: {
        Type: "AWS::CloudFront::OriginRequestPolicy",
        Properties: {
          OriginRequestPolicyConfig: {
            Name: 'SvelteKitSsrApiGatewayOriginRequestPolicy',
            HeadersConfig: { HeaderBehavior : 'none' },
            QueryStringsConfig: { QueryStringBehavior: 'all' },
            CookiesConfig: { CookieBehavior: 'all' }
          }
        }
      },

      SvelteKitDistribution: {
        Type: 'AWS::CloudFront::Distribution',
        Properties: {
          DistributionConfig: {
            Comment: name,
            Enabled: true,
            HttpVersion: 'http2',
            PriceClass: 'PriceClass_200',
            DefaultCacheBehavior: {
              TargetOriginId: 'SvelteKitSsrApiGatewayOrigin',
              Compress: true,
              ViewerProtocolPolicy: 'redirect-to-https',
              AllowedMethods: [ 'GET', 'HEAD', 'OPTIONS', 'PUT', 'PATCH', 'POST', 'DELETE' ],
              CachePolicyId: '4135ea2d-6df8-44a3-9df3-4b5a84be39ad', // Managed-CachingDisabled
              OriginRequestPolicyId: { Ref: 'SvelteKitSsrApiGatewayOriginRequestPolicy' }
            },
            CacheBehaviors: [
              {
                TargetOriginId: 'SvelteKitStaticContentsBucketOrigin',
                PathPattern: '/_app/*',
                Compress: true,
                ViewerProtocolPolicy: 'redirect-to-https',
                AllowedMethods: [ 'GET', 'HEAD' ],
                CachePolicyId: '658327ea-f89d-4fab-a63d-7e88639e58f6' // Managed-CachingOptimized
              }
            ],
            Origins: [
              {
                Id: 'SvelteKitSsrApiGatewayOrigin',
                DomainName: { "Fn::Join": [ ".", [{ Ref: 'HttpApi' }, 'execute-api', { Ref: 'AWS::Region' }, { Ref: 'AWS::URLSuffix' }]]},
                CustomOriginConfig: {
                  OriginProtocolPolicy: 'https-only'
                }
              },
              {
                Id: 'SvelteKitStaticContentsBucketOrigin',
                DomainName: { 'Fn::GetAtt': [ 'SvelteKitStaticContentsBucket', 'DomainName' ]},
                S3OriginConfig: {
                  OriginAccessIdentity: { 'Fn::Join': [ '/', [ 'origin-access-identity/cloudfront', { Ref: 'SvelteKitCloudFrontOriginAccessIdentity' }]]}
                }
              }
            ]
          }
        }
      }
    }
  }
}))();
```
:::


#### serverless-s3-deploy の設定
デプロイを実行した際に `build` ディレクトリ(= SvelteKit の静的コンテンツの全ファイル)を S3 へアップロードします。`empty: true` でアップロード時に一度 S3 を空にしてます。

```typescript:serverless-config.ts
  plugins: [
    'serverless-s3-deploy'
  ],

  custom: {
    assets: {
      auto: true,
      targets: [{
        bucket: { Ref: 'SvelteKitStaticContentsBucket' },
        empty: true,
        files: [{
          source: 'build',
          globs: '**'
        }]
      }]
    }
  },
```


#### Lambda の設定
API Gateway の HTTP API で全リクエストを Adapter で生成した Lambda へ送ります。

```typescript:serverless-config.ts
  package: {
    individually: true,
    patterns: [
      '!**',
      '.aws-apigw/**'
    ]
  },

  functions: {
    sveltekit: {
      handler: '.aws-apigw/functions-internal/__render.handler',
      events: [
        { httpApi: { method: '*', path: '*' }}
      ]
    }
  },
```


#### CloudFormation で S3 の設定
静的コンテンツを保持する S3 Bucket と、CloudFront からアクセスを許可するポリシーです。

```typescript:serverless-config.ts
      SvelteKitStaticContentsBucket: {
        Type: 'AWS::S3::Bucket', Properties: {}
      },

      SvelteKitStaticContentsBucketPolicy: {
        Type: 'AWS::S3::BucketPolicy',
        Properties: {
          Bucket: { Ref: 'SvelteKitStaticContentsBucket' },
          PolicyDocument: {
            Statement: [{
              Effect: 'Allow',
              Action: [ 's3:GetObject' ],
              Resource: { 'Fn::Join': [ '/', [{ 'Fn::GetAtt': [ 'SvelteKitStaticContentsBucket', 'Arn' ]}, '*' ]]},
              Principal: {
                CanonicalUser: { 'Fn::GetAtt': [ 'SvelteKitCloudFrontOriginAccessIdentity', 'S3CanonicalUserId' ]}
              }
            }]
          }
        }
      },
```


#### CloudFront からのアクセス設定
CloudFront の Origin アクセスの設定です。
- S3 へは `CloudFrontOriginAccessIdentity` でアクセスします
- API Gateway へは OAuth2 フローで付与される Query と、sk-auth が認証後に設定する Cookie を Lambda へ渡す必要があり、その設定の `OriginRequestPolicy` を作ります

```typescript:serverless-config.ts
      SvelteKitCloudFrontOriginAccessIdentity: {
        Type: 'AWS::CloudFront::CloudFrontOriginAccessIdentity',
        Properties: {
          CloudFrontOriginAccessIdentityConfig: { Comment: name }
        }
      },

      SvelteKitSsrApiGatewayOriginRequestPolicy: {
        Type: "AWS::CloudFront::OriginRequestPolicy",
        Properties: {
          OriginRequestPolicyConfig: {
            Name: 'SvelteKitSsrApiGatewayOriginRequestPolicy',
            HeadersConfig: { HeaderBehavior : 'none' },
            QueryStringsConfig: { QueryStringBehavior: 'all' },
            CookiesConfig: { CookieBehavior: 'all' }
          }
        }
      },
```


#### CloudFront の設定
最後に CloudFront で S3 と API Gateway への振り分けを行います。大事なポイントです。

設定と逆順になりますが実際の動きをイメージしやすい順に説明します。
まず CloudFront がアクセスする API Gateway と S3 の `Origins` 定義をします。

次に `CacheBehaviors` の `SvelteKitStaticContentsBucketOrigin` を定義します。これは SvelteKit の静的コンテンツへのアクセス(= `/_app/*`) を S3 Origin へ回すためのものです。またキャッシュを活用するため AWS マネージドのキャッシュ・ポリシーを適用しています。

最後に `DefaultCacheBehavior` で `SvelteKitStaticContentsBucketOrigin` へ当てはまらなかったリクエストを、すべて API Gateway(= Lambda) へ送ります。設定ンポイントは以下です。
- OAuth2 フローで `HTTP POST` が来るので `AllowedMethods` で許可
- Query と Cookie を転送するように先ほど定義した `OriginRequestPolicyId` を適用
- キャッシュは不要なためマネージドのキャッシュ無効ポリシーを適用

```typescript:serverless-config.ts
      SvelteKitDistribution: {
        Type: 'AWS::CloudFront::Distribution',
        Properties: {
          DistributionConfig: {
            Comment: name,
            Enabled: true,
            HttpVersion: 'http2',
            PriceClass: 'PriceClass_200',
            DefaultCacheBehavior: {
              TargetOriginId: 'SvelteKitSsrApiGatewayOrigin',
              Compress: true,
              ViewerProtocolPolicy: 'redirect-to-https',
              AllowedMethods: [ 'GET', 'HEAD', 'OPTIONS', 'PUT', 'PATCH', 'POST', 'DELETE' ],
              CachePolicyId: '4135ea2d-6df8-44a3-9df3-4b5a84be39ad', // Managed-CachingDisabled
              OriginRequestPolicyId: { Ref: 'SvelteKitSsrApiGatewayOriginRequestPolicy' }
            },
            CacheBehaviors: [
              {
                TargetOriginId: 'SvelteKitStaticContentsBucketOrigin',
                PathPattern: '/_app/*',
                Compress: true,
                ViewerProtocolPolicy: 'redirect-to-https',
                AllowedMethods: [ 'GET', 'HEAD' ],
                CachePolicyId: '658327ea-f89d-4fab-a63d-7e88639e58f6' // Managed-CachingOptimized
              }
            ],
            Origins: [
              {
                Id: 'SvelteKitSsrApiGatewayOrigin',
                DomainName: { "Fn::Join": [ ".", [{ Ref: 'HttpApi' }, 'execute-api', { Ref: 'AWS::Region' }, { Ref: 'AWS::URLSuffix' }]]},
                CustomOriginConfig: {
                  OriginProtocolPolicy: 'https-only'
                }
              },
              {
                Id: 'SvelteKitStaticContentsBucketOrigin',
                DomainName: { 'Fn::GetAtt': [ 'SvelteKitStaticContentsBucket', 'DomainName' ]},
                S3OriginConfig: {
                  OriginAccessIdentity: { 'Fn::Join': [ '/', [ 'origin-access-identity/cloudfront', { Ref: 'SvelteKitCloudFrontOriginAccessIdentity' }]]}
                }
              }
            ]
          }
        }
```

※ CloudFront のマネージド・ポリシーの詳細はこちら
　 [管理キャッシュポリシーの使用 - Amazon CloudFront](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/using-managed-cache-policies.html)



### TypeScript (ts-node) の設定
SvelteKit は `es2020` モジュールでビルドされ、いっぽうで Serverless は CommonJS です。
Serverless を CommonJS として動くように `moduleTypes` と `include` の設定を追加します。

```json:tsconfig.json
{
  ...省略
  "ts-node": {
    "moduleTypes": {
      "serverless-config.ts": "cjs"
    }
  },
  "include": ["serverless-config.ts", "src/**/*.d.ts", "src/**/*.js", "src/**/*.ts", "src/**/*.svelte"]
}
```


## 認証実装を CloudFront 用に修正
SSR は Lambda で CloudFront とは別ドメインのホストで動きます。
具体的には Lambda 側のホストは `.execute-api.[region].amazonaws.com` となり、CloudFront 側は `.cloudfront.net` です。このままだと sk-auth が生成する `redirect_uri` は Lambda のホストのドメインとなり OAuth2 フローで CloudFront へ帰れなくなります。

`redirect_uri` が CloudFront のドメインを指すように修正します。
`src/lib/auth.ts` の GoogleOAuth2Provider インスタンスを作っているところを、以下のように継承したクラスで `getCallbackUri()` をオーバーライドし環境変数から設定された値を返します。

```typescript:src/lib/auth.ts
import { Providers, SvelteKitAuth } from 'sk-auth';

export const auth = new SvelteKitAuth({
  jwtSecret: import.meta.env.VITE_JWT_SECRET,
  providers: [
    new class extends Providers.GoogleOAuth2Provider {

      constructor() {
        super({
          clientId: import.meta.env.VITE_GOOGLE_OAUTH_CLIENT_ID,
          clientSecret: import.meta.env.VITE_GOOGLE_OAUTH_CLIENT_SECRET,
          profile(profile) {
            return { ...profile, provider: 'google' };
          }
        });
      }

      getCallbackUri(): string {
        return import.meta.env.VITE_CALLBACK_URI_FOR_CLOUDFRONT;
      }
    }()
  ]
});
```

環境変数の定義が増えたので `VITE_CALLBACK_URI_FOR_CLOUDFRONT` の型定義も追加します。
```typescript:src/global.d.ts
/// <reference types="@sveltejs/kit" />

interface ImportMetaEnv {
  VITE_JWT_SECRET: string;
  VITE_GOOGLE_OAUTH_CLIENT_ID: string;
  VITE_GOOGLE_OAUTH_CLIENT_SECRET: string;
  VITE_CALLBACK_URI_FOR_CLOUDFRONT: string;
}
```

環境変数 `VITE_CALLBACK_URI_FOR_CLOUDFRONT` の値を `.env` に追加します。
CloudFront のドメインを入れるので一度デプロイしてから再設定してデプロイしなおします。
※ 同じく Google Cloud Platform の「認証情報」のリダイレクト URI も修正しておきます
```shell:.env
# JWT secret used to sign and verify the cookies (256-bits or more)
VITE_JWT_SECRET=[your-jwt-secret-over-256-bit]

# Client ID for Google OAuth
VITE_GOOGLE_OAUTH_CLIENT_ID=[your-google-oauth-client-id]

# Client Secret for Google OAuth
VITE_GOOGLE_OAUTH_CLIENT_SECRET=[your-google-oauth-client-secret]

# Callback URI for CloudFront with CloudFront/API Gateway integration
VITE_CALLBACK_URI_FOR_CLOUDFRONT=https://[your-cloudfront-domain]/api/auth/callback/google
```

:::message alert
.env ファイルを GitHub へコミットしないでください
.gitignore へ追加するなどして対策をしてください
:::


## デプロイと動作確認
デプロイ・コマンドを npm scripts、`package.json` の `scripts` へ追加します。

```json:package.json
{
  ...(省略)
  "scripts": {
    ...(省略)
    "deploy": "serverless deploy --config serverless-config.ts --verbose"
  }
}
```

`npm run deploy` を実行することでデプロイされます。

初回は CloudFront がデプロイされるため、かなり時間がかかります。コンソールで進捗がないように感じてもデプロイは実行されているので気長に待ってください。

コマンド実行後、コンソール出力の後半に Stack Outputs の `SvelteKitDistribution` が出力されます。これが CloudFront のドメインで .env や認証設定のリダイレクト URI へ設定します。
.env 設定後に再デプロイし、このドメインへブラウザでアクセスすると動作確認できます。

```shell-session
$ npm run deploy

> sveltekit-google-auth-aws-apigw@0.0.1 deploy
> serverless deploy --config serverless-config.ts --verbose

Serverless: Packaging service...
...(省略)

Stack Outputs
SvelteKitDistribution: a10qz7j3ikaaXX.cloudfront.net
HttpApiId: a1rpa7y7XX
ServerlessDeploymentBucketName: sveltekit-google-auth-aw-serverlessdeploymentbuck-a23ajz9jgnidXX
HttpApiUrl: https://a2rpad7y7XX.execute-api.us-east-1.amazonaws.com

Serverless: Emptying bucket: sveltekit-google-auth-aw-sveltekitstaticcontentsb-a1zds7ivdsyjXX
Serverless: Sync bucket: sveltekit-google-auth-aw-sveltekitstaticcontentsb-a1zds7ivdsyjXX
Serverless: Path: build
Serverless:     File:  _app/assets/start-61d1577b.css (text/css)
...(省略)
```


## まとめ
SvelteKit の SSR を AWS にサーバーレスな構成で構築することができました。

サーバーレスにすることでコンテナーやインスタンスの管理から解放され、純粋にアプリの開発とメンテに注力できるのが嬉しいですね。

Lambda 用の SvelteKit Adapter は、Netlify Adapter の流用で手軽に作ることができます。モジュール化して公開できるとよいのですが、SvelteKit 自体が開発中であり、また Adapter の API は変わる可能性があるとの注意書きがあることから、当面はローカル実装かなと思います。


## 参考サイト
- [Svelte • サイバネティクスで強化されたWebアプリ](https://svelte.jp/)
- [SvelteKit • The fastest way to build Svelte apps](https://kit.svelte.dev/)
- [Dan6erbond/sk-auth](https://github.com/Dan6erbond/sk-auth)
- [adapter-netlify](https://github.com/sveltejs/kit/tree/master/packages/adapter-netlify)
- [Serverless - AWS Documentation](https://www.serverless.com/framework/docs/providers/aws)
