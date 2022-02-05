---
title: "Day.js で「〇〇分間」のような期間の日時を扱う"
emoji: "⌛"
type: "tech"
topics: [ tips, dayjs, javascript, typescript, nodejs ]
published: true
announce:
  記事を書きました。
  「24時間」や「５分間」のような期間を表すような時間を扱うことがあります。内部的にはミリ秒でよいのですが画面表示となると「日時分秒」などの単位計算が必要です。便利な Day.js の Tips を紹介。

  ⇒ Day.js で「〇〇分間」のような期間の日時を扱う
  https://zenn.dev/lulzneko/articles/handles-duration-datetime-in-dayjs
---

## はじめに
「24時間」や「5分間」などのように、ある期間のような日時を扱いたいことがあります。

そのような場合は１日 = `24 * 60 * 60 * 1000` のようなミリ秒で保持していたりします。
計算するうえではミリ秒を使うと便利ですが、これを画面の表示などに使うとすると「日時分秒」などの単位を求める必要があります。

このようなケースでも Day.js は Plugin で対応してくれます。
前回に続き Day.js の Tips を紹介します。
⇒ [Day.js で「〇〇分前」のような相対日時を扱う](https://zenn.dev/lulzneko/articles/handles-relative-datetime-in-dayjs)

### 解決したい課題
- 時間が絶対表記で分かりにくい
- 期間の時間表記の実装がたいへん
- 時間の丸目処理がたいへん

### 動作環境
本記事は以下の環境で動作を確認しています。
- Node.js 16
- Day.js 1.10.7
- TypeScript: 4.5
- Ubuntu 18.04 LTS (WSL1)


## Day.js の Duration plugin
公式の Duration プラグインを使うことで簡単に実装できます。
https://day.js.org/docs/en/plugin/duration

12ヶ月の期間を表現するシンプルなコード例です。
```typescript: TypeScript
import dayjs, { extend } from 'dayjs';
import duration from 'dayjs/plugin/duration';

extend(duration);

dayjs.duration({ months: 12 });
```

期間を扱うプラグイン duration をインポートします。
`extend(duration);` で dayjs がプラグインを使えるように登録します。

`dayjs.duration()` で扱いたい期間を作ります。
引数は上記例のように期間を表すオブジェクトのほかに、ミリ秒、文字列などを受け取ります。
よく使われるケースとしてはミリ秒(e.g. `24 * 60 * 60 * 1000`)でしょうか。

変換もサポートされていますが、ミリ秒で指定しないと若干ずれるようです。

```javascript: 実行例
> dayjs.duration({ months: 12 }).asYears();
0.9863013698630136

> dayjs.duration({ months: 12 }).asDays();
360

> dayjs.duration(365 * 24 * 60 * 60 * 1000).asYears();
1

> dayjs.duration(365 * 24 * 60 * 60 * 1000).asDays();
365
```


## ヒューマナイズ表記
期間の値だけでしたらエポック・ミリ秒(UNIX ミリ秒)で扱うでもよいでしょう。
Day.js は人間にとって読みやすい単位に丸めてくれるところです。素晴らしい！
https://day.js.org/docs/en/durations/humanize

```typescript: TypeScript
import dayjs, { extend } from 'dayjs';
import duration from 'dayjs/plugin/duration';
import relativeTime from 'dayjs/plugin/relativeTime';

extend(duration);
extend(relativeTime);

dayjs.duration({ months: 12 }).humanize();
```

[RelativeTime](https://day.js.org/docs/en/plugin/relative-time)に依存しているので、Duration に加えて RelativeTime も読み込みます。
Duration のインスタンスに対して `.humanize()` を呼び出します。

```javascript: 実行例
> dayjs.duration({ months: 12 }).humanize();
'a year'

> dayjs.duration(365 * 24 * 60 * 60 * 1000).humanize();
'a year'
```

引数 `withSuffix` を受け取り、「in」を付ける場合は true で、デフォルト false。

```javascript: 実行例
> dayjs.duration({ months: 12 }).humanize();
'a year'

> dayjs.duration({ months: 12 }).humanize(true);
'in a year'
```

また、負の数も扱えます。

```javascript: 実行例
> dayjs.duration({ months: -12 }).humanize();
'a year'

> dayjs.duration({ months: -12 }).humanize(true);
'a year ago'
```


## 日本語の表記
[Day.js の i18n](https://day.js.org/docs/en/i18n/i18n) に対応しているため、通常の日本語対応で表記が変わります。

```typescript: TypeScript
import 'dayjs/locale/ja';
import dayjs, { locale, extend } from 'dayjs';
import duration from 'dayjs/plugin/duration';
import relativeTime from 'dayjs/plugin/relativeTime';

locale('ja');
extend(duration);
extend(relativeTime);

dayjs.duration({ months: 12 }).humanize();
```

```javascript: 実行例
dayjs.duration({ months: 12 }).humanize();
'1年'

dayjs.duration({ months: 12 }).humanize(true);
'1年後'

dayjs.duration({ months: -12 }).humanize(true);
'1年前'
```


## まとめ
Day.js を使うことで、期間を表す時間の表記が簡単に扱えます。
そして Day.js が i18n 対応しているため、相対表記も設定に合わせて表記言語が変わります。

期間自体はエポックでも十分ですが、表示に必要な丸め処理は Day.js が助けてくれます。

## 参考サイト
- [Day.js · 2kB JavaScript date utility library](https://day.js.org/)
- [iamkun/dayjs: ⏰ Day.js 2kB immutable date-time library alternative to Moment.js with the same modern API](https://github.com/iamkun/dayjs/)
- [Duration · Day.js](https://day.js.org/docs/en/plugin/duration)
- [Humanize · Day.js](https://day.js.org/docs/en/durations/humanize)
