---
title: "GitHub とかでユーザー登録時の初期アイコンのアレ"
emoji: "🎨"
type: "tech"
topics: [ identicon, icon, svg, nodejs, javascript ]
published: true
announce:
  記事を書きました。
  GitHub などに登録した際に表示される初期アイコン。その名称と生成してくれるライブラリーを紹介。

  自分のアプリやサイトに簡単で綺麗な初期表示アイコン生成を導入することができます。

  ⇒ GitHub とかでユーザー登録時の初期アイコンのアレ
  https://zenn.dev/lulzneko/articles/initial-icon-when-user-registered-github-or-other
---

## はじめに
GitHub へユーザー登録した際に、ユーザーごとにランダムな初期アイコンが生成されます。
![GitHub Identicon](https://github.com/identicons/lulzneko.png =80x)

色だけではなく模様もランダムになるので、初期アイコンから変えていなくても識別できるのでとても助かります。シンプルでありながらもデザイン性もあり、Organization や Team など明示的なアイコンを付けないケースでもしっくりきたり、愛着がでてきたりして不思議なものです。

その不思議な「アレ」、何という名称か知ってますか。知りませんでした。。。

初期アイコンの生成について調べたので紹介します。

### 解決したい課題
- GitHub とかの初期生成アイコンの名称が分からない
- ユーザー発行時の初期アイコンに困っている
- 初期アイコンの種類が用意できない

### 動作環境
本記事は以下の環境で動作を確認しています。
- Node.js 16
- Ubuntu 18.04 LTS (WSL1)


## その名は「Identicon」
最初に名称ですが「Identicon」と言うそうです。

Wikipedia によると [Identicon](https://en.wikipedia.org/wiki/Identicon) は、2007年 Don Park さん考案で IP アドレスやユーザー他、さまざまな情報を視覚的に楽しく識別するためのものとのことです。

![Visual Security: 9-block IP Identification](/images/identicon.png)

それよりも前には 1999年の Adrian Perrig さん、Dawn Song さん考案の SSH キーのランダムアート(今も表示されるアレ)があるとのことで歴史あるものなんですね。
```
The key's randomart image is:
+--[ED25519 256]--+
|  ..o.=.  .o.  .+|
| ... E + .=o+ .. |
|o.. o + B.oO  .. |
|.+   o = *=....  |
|  + o o S ++.. . |
| . . + . ...  .  |
|    . o . .      |
|     .   .       |
|                 |
+----[SHA256]-----+
```

GitHub には 2013年に導入されたようです。
https://github.blog/2013-08-14-identicons/

※ GitHub の Identicon を調べるには、上記記事の URL ではなく以下の URL の `[username]` を調べたいユーザー名にしてアクセスします。自分の Identicon が何だったのかも調べられます。
`https://github.com/identicons/[username].png`



## Identicon 生成ライブラリー「Jdenticon」
初期アイコンの不思議なアレが Identicon だと分かったところで、自前のアプリなどにも組み込みたいところです。その Identicon を簡単に生成してくれるのが「[Jdenticon](https://jdenticon.com/)」です。

![Jdenticon](/images/jdenticon.png)

適当な文字列を与えると Identicon の SVG を生成してくれます。基本的には保存する必要はなくユーザーアイコンがあれば <img> タグを表示し、なければ Jdenticon で SVG を生成して表示するという、実行時ベースでの使い方になります。


### HTML での実装
公式のコード例から抜粋します。

```html
<!-- Using jsDelivr -->
<script src="https://cdn.jsdelivr.net/npm/jdenticon@3.1.1/dist/jdenticon.min.js"
        integrity="sha384-l0/0sn63N3mskDgRYJZA6Mogihu0VY3CusdLMiwpJ9LFPklOARUcOiWEIGGmFELx"
        crossorigin="anonymous">
</script>

<!-- Vector icon -->
<svg width="80" height="80" data-jdenticon-value="[hash]"></svg>
```

まずは CDN から Jdenticon のライブラリーをロードします。

次に表示したい場所に <svg> タグを記述し `data-jdenticon-value` 属性にユーザー ID やハッシュなどの識別文字列を入れます。`data-jdenticon-value` の値で Identicon が変わります。


### npm を使っての組み込みでの実装
もう少し実践的に npm で UI フレームワークに組み込んだ例を紹介します。

まずは jdenticon をインストールします。
```shell-session
npm i jdenticon
```

次に利用する箇所でインポートとスクリプトを記述します。
以下は [Svelte](https://svelte.jp/) での例ですが、各フレームワークの実装ポイントに同様に記述します。
```html
<!-- Using jsDelivr -->
<script lang="ts">
  import { toSvg } from 'jdenticon';
</script>

<div>
  {@html toSvg('[hash]', 32)}
</div>
```

まずは jdenticon の `toSvg` をインポートします。

続いて表示する箇所で `toSvg('[hash]', [size])` として呼び出します。`[hash]` はユーザー ID やハッシュなどの識別文字列で、`[size]` は SVG の大きさをピクセルで指定します。

<svg> タグが生成されるので HTML タグとして機能するようにビューへ出力します。Svelte では `{@html}` ですし、それぞれのフレームワークで用意されているので、それを使います。


## まとめ
GitHub などでユーザー登録した際に初期生成されるアイコンは「Identicon」と呼ばれます。

そして Jdenticon を使うことでアプリなどに簡単に Identicon を導入できます。
実行時に綺麗な SVG を生成でき、リソース管理も不要なのでとても助かります。

また Jdenticon は [Icon designer](https://jdenticon.com/icon-designer.html) も用意されておりカラーパレットのカスタマイズなども視覚的にできるのでアプリやサイトのテイストに合わせた設定を手軽に作れます。


**2022.01.20 追記**
かわいい顔アイコンの Identicon を生成するライブラリー紹介を書きました。
https://zenn.dev/lulzneko/articles/library-for-generating-cute-face-icon-identicons

## 参考サイト
- [Identicon - Wikipedia](https://en.wikipedia.org/wiki/Identicon)
- [Identicons! | The GitHub Blog](https://github.blog/2013-08-14-identicons/)
- [Jdenticon - Open source identicon generator](https://jdenticon.com/)
