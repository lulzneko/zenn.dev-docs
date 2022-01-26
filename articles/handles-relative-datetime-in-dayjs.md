---
title: "Day.js で「〇〇分前」のような相対日時を扱う"
emoji: "⏳"
type: "tech"
topics: [ tips, dayjs, javascript, typescript, nodejs ]
published: true
announce:
  記事を書きました。
  ソーシャルメディアなどの更新日時で使われる「〇〇分前」のような相対時間での表記
  Day.js を使うことで簡単に作ることができ、また Day.js が i18n 対応してくれているので日本語表記も簡単にできます

  ⇒ Day.js で「〇〇分前」のような相対日時を扱う
  https://zenn.dev/lulzneko/articles/handles-relative-datetime-in-dayjs
---

## はじめに
ソーシャルメディアなどではコンテンツの作成時間や更新時間が「〇〇分」や「〇〇分前」などのように表記されています。ソーシャルに限らず増えてきたようにも感じます。

タイムライン形式の UI を作っていて「〇〇分前」のような表示をしたいが、その言葉が分からない。例によって「Twitter とかで時間を表示している〇〇分前とかの『アレ』」を探すことに。言葉さえわかればわけないのですが、いかんせん「〇〇分前って表示するアレ」としか。。。
(前回の「アレ」 ⇒ [GitHub とかでユーザー登録時の初期アイコンのアレ](https://zenn.dev/lulzneko/articles/initial-icon-when-user-registered-github-or-other))

そんな話を一緒に開発している同僚にしたところ、いくつかのライブラリーを見つけてくれました。それらの要素としては「Relative Time」や「ago time」、つまり『相対時間』。やっと「アレ」を脱却できました。(ありがと！)

そのような相対日時の表記をするためのライブラリーがいくつもあったのですが、普段からよく使っている [Day.js](https://github.com/iamkun/dayjs) がデフォルトで対応していました。

Day.js で「〇〇分前」のような相対表記と、日本語での表記対応について Tips を紹介します。


### 解決したい課題
- 時間が絶対表記で分かりにくい
- 時間の相対表記の実装がたいへん

### 動作環境
本記事は以下の環境で動作を確認しています。
- Node.js 16
- Day.js 1.10.7
- TypeScript: 4.5
- Ubuntu 18.04 LTS (WSL1)
- Google Cloud Platform (OAuth 2.0)


## Day.js の RelativeTime plugin
公式の RelativeTime プラグインを使うことで簡単に実装できます。
https://day.js.org/docs/en/plugin/relative-time

現在時刻からの相対時間を取得するシンプルなコード例です。
```typescript: TypeScript
import dayjs, { extend } from 'dayjs';
import relativeTime from 'dayjs/plugin/relativeTime';

extend(relativeTime);

dayjs().fromNow();
```

相対時間を扱うプラグイン relativeTime をインポートします。
`extend(relativeTime);` で dayjs がプラグインを使えるように登録します。

あとは相対時間を扱いたい日時の dayjs インスタンスに対して `.fromNow()` を呼び出します。
(ここでは `dayjs()`、現在日時なので実質的な相対時間はわずか、`a few seconds ago` ですが)


## その他の RelativeTime API
現在との相対時間の他に、以下の API が提供されています。

### 現在日時からの相対日時を返す
`.fromNow(withoutSuffix?: boolean)`

先のコード例に上げた現在時刻からの相対日時を返します。
引数 `withoutSuffix` は、返す文字列の末尾「ago」を消す場合は true で、デフォルト false。

```javascript: 実行例
> dayjs('2000/01/01').fromNow();
'22 years ago'

> dayjs('2000/01/01').fromNow(true);
'22 years'
```


### 指定日時からの相対日時を返す
`.from(compared: Dayjs, withoutSuffix?: boolean)`

第一引数で指定した日時からの相対日時を返します。

```javascript: 実行例
> dayjs('2000/01/01').from('2022/01/01');
'22 years ago'
```


### 現在日時への相対時間を返す
`.toNow(withoutSuffix?: boolean)`

表現が「ago」から「in」に変わります。
日本語だと「〇〇分前」が「〇〇分後」のように「後」の表記になります。

引数 `withoutSuffix` は、返す文字列の「in」を消す場合は true で、デフォルト false。

```javascript: 実行例
> dayjs('2000/01/01').toNow();
'in 22 years'

> dayjs('2000/01/01').toNow(true);
'22 years'
```


### 指定日時への相対時間を返す
`.to(compared: Dayjs, withoutSuffix?: boolean)`

第一引数で指定した日時への相対日時を返します。

```javascript: 実行例
> dayjs('2000/01/01').toNow('2022/01/01');
'22 years'
```


## 日本語の表記
[Day.js の i18n](https://day.js.org/docs/en/i18n/i18n) に対応しているため、通常の日本語対応で表記が変わります。

```typescript: TypeScript
import 'dayjs/locale/ja';
import dayjs, { locale, extend } from 'dayjs';
import relativeTime from 'dayjs/plugin/relativeTime';

locale('ja');
extend(relativeTime);

dayjs().fromNow();
```

```javascript: 実行例
> dayjs('2000/01/01').fromNow();
'22年前'
```


## まとめ
Day.js を使うことで、簡単に相対時間の表記が扱えます。
そして Day.js が i18n 対応しているため、相対表記も設定に合わせて表記言語が変わります。

意外と身近なライブラリーでサポートしてくれていたんですね。助かります。

## 参考サイト
- [Day.js · 2kB JavaScript date utility library](https://day.js.org/)
- [iamkun/dayjs: ⏰ Day.js 2kB immutable date-time library alternative to Moment.js with the same modern API](https://github.com/iamkun/dayjs/)
- [RelativeTime · Day.js](https://day.js.org/docs/en/plugin/relative-time)
