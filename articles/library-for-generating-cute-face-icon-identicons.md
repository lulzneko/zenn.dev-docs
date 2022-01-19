---
title: "かわいい顔アイコンな Identicon を生成するライブラリー"
emoji: "🙂"
type: "tech"
topics: [ identicon, icon, svg, nodejs, javascript ]
published: true
announce:
  記事を書きました。
  GitHub などに登録した際に表示される初期アイコン、Identicon

  幾何学模様の Identicon はカッコよくて素敵ですが、ちょっと緩い感じのかわいい顔を生成するライブラリーがありましたので紹介

  ⇒ かわいい顔アイコンな Identicon を生成するライブラリー
  https://zenn.dev/lulzneko/articles/library-for-generating-cute-face-icon-identicons
---

## はじめに
GitHub などにユーザー登録した際にランダムな初期アイコンが設定されます。

そのようなランダムなアイコンは Identicon と呼ばれ、生成するためのライブラリーもあることを「[GitHub とかでユーザー登録時の初期アイコンのアレ](https://zenn.dev/lulzneko/articles/initial-icon-when-user-registered-github-or-other)」で紹介しました。

その Identicon 生成ライブラリーで「かわいい顔アイコン」を生成してくれるライブラリーがありましたので紹介します。


## Boring Avatars
Boring Avatars は６種類の Identicon (サイトではアバターと表記)が生成できる React のライブラリーです。
https://github.com/boringdesigners/boring-avatars

そのうちの１つ `beam` がシンプルでかわいい顔を生成してくれます。
![Beam - Boring Avatars](/images/boring-avatars.png)


## 使い方
公式サイトからの抜粋ですが、以下の手順で簡単に設置できます。
React のライブラリーと謳っているとおり React で動作します。

npm をインストールします。

```shell-session
npm install boring-avatars
```

Identicon を設置する場所で以下のように記述します。

```javascript
import Avatar from "boring-avatars";

<Avatar
  size={40}
  name="Maria Mitchell"
  variant="beam"
  colors={["#92A1C6", "#146A7C", "#F0AB3D", "#C271B4", "#C20D90"]}
/>;
```

設定項目は以下です。

| 項目    | 設定値や用途                                                                   |
|:--------|:-------------------------------------------------------------------------------|
| size    | <svg> タグの `width` と `height`、ピクセルで指定                               |
| square  | Identicon を四角形で表示する場合 true、円形は false                            |
| name    | Identicon 生成の文字列、同じ文字列は同じ絵柄になるので ID など一意なものがよい |
| variant | 顔アイコンは `beam`、他にも `pixel`、`sunset`、`ring`、`bauhaus` が指定できる  | 
| colors  | カラーパレットで指定が必要、<path> や <rect> の `fill` や `stroke` に使用      |


## カラーパレットと Playground
設定項目の `colors` は必須項目で、自前で用意する必要があります。

完全ランダムでもよさそうですが、幾何学模様の Identicon と違って顔の場合はカラーパレットを指定しないと難しかったのでしょうか。このカラーパレットを考えるのが意外と難しいので、公式が用意してくれた Playground を使うと便利です。

https://boringavatars.com/

Playground には、たくさんの表情が並んでいるのでカラーパレットによる雰囲気が分かりやすいです。画面上部中央に５色のカラーパレットがありクリックすることで変更できます。

また画面右上の「Random palette」ボタンを押すと５色のカラーパレットをランダムで生成します。他の Identicon の種類や、四角形への変更、ダークモードでの表示など色々と試せます。


## React 以外のポート
まずは公式として React 依存を分離して、さまざまなフレームワークへの対応を始めようというプルリクエストがあがっています。公式対応されると嬉しいですね。
https://github.com/boringdesigners/boring-avatars/pull/18

Issues を見ていくと以下のポートが紹介されています。
- [paolotiu/svelte-boring-avatars](https://github.com/paolotiu/svelte-boring-avatars)
- [luhart/react-native-boring-avatars](https://github.com/luhart/react-native-boring-avatars)
- [khaled-sadek/blade-boring-avatars](https://github.com/khaled-sadek/blade-boring-avatars)
- [mujahidfa/vue-boring-avatars](https://github.com/mujahidfa/vue-boring-avatars)
- [stonega/vue2-boring-avatars](https://github.com/stonega/vue2-boring-avatars)
- [MitchellCash/ng-boring-avatars](https://github.com/MitchellCash/ng-boring-avatars)

また API から Identicon を取得することもできます。
あまり負荷をかけすぎるのもよくないので、基本的には組み込みを使った方がよいでしょう。
[https://source.boringavatars.com/beam/120/Maria%20Mitchell?colors=264653,2a9d8f,e9c46a,f4a261,e76f51](https://source.boringavatars.com/beam/120/Maria%20Mitchell?colors=264653,2a9d8f,e9c46a,f4a261,e76f51)

実装としては React に深く依存してないので、必要なタイプをポートするのは簡単です。<svg> タグを作っている部分の `{}` と `()` をテンプレートリテラルや文字列連結に変えるだけです。

beam タイプ(v1.6.1)では以下のコード範囲です。
[https://github.com/boringdesigners/boring-avatars/blob/v1.6.1/src/lib/components/avatar-beam.js#L40-L127](https://github.com/boringdesigners/boring-avatars/blob/v1.6.1/src/lib/components/avatar-beam.js#L40-L127)

参考までに v1.6.1 の beam を TypeScript & テンプレートリテラルにしたものを貼ります。

:::details boring-avatars-beam.ts
```typescript
// Boring Avatars - Beam
// https://github.com/boringdesigners/boring-avatars/blob/v1.6.1/src/lib/components/avatar-beam.js

const SIZE = 36;

type Props = {
  colors: string[];
  name: string;
  square: boolean;
  size: number;
};

export const avatarBeam = (props: Props): string => {
  const data = generateData(props.name, props.colors);
  return `
    <svg
      viewBox="0 0 ${SIZE} ${SIZE}"
      fill="none"
      role="img"
      xmlns="http://www.w3.org/2000/svg"
      width=${props.size}
      height=${props.size}
    >
      <mask id="mask__beam" maskUnits="userSpaceOnUse" x=${0} y=${0} width=${SIZE} height=${SIZE}>
        <rect width=${SIZE} height=${SIZE} rx=${props.square ? undefined : SIZE * 2} fill="#FFFFFF" />
      </mask>
      <g mask="url(#mask__beam)">
        <rect width=${SIZE} height=${SIZE} fill=${data.backgroundColor} />
        <rect
          x="0"
          y="0"
          width=${SIZE}
          height=${SIZE}
          transform="
            translate(${data.wrapperTranslateX} ${data.wrapperTranslateY})
            rotate(${data.wrapperRotate} ${SIZE / 2} ${SIZE / 2})
            scale(${data.wrapperScale})
          "
          fill=${data.wrapperColor}
          rx=${data.isCircle ? SIZE : SIZE / 6}
        />
        <g
          transform="
            translate(${data.faceTranslateX} ${data.faceTranslateY})
            rotate(${data.faceRotate} ${SIZE / 2} ${SIZE / 2})
          "
        >
          ${data.isMouthOpen
            ? `<path d="M15 ${19 + data.mouthSpread}c2 1 4 1 6 0" stroke=${data.faceColor} fill="none" strokeLinecap="round" />`
            : `<path d="M13, ${19 + data.mouthSpread} a1, 0.75 0 0, 0 10, 0" fill=${data.faceColor} />`
          }
          <rect
            x=${14 - data.eyeSpread}
            y=${14}
            width=${1.5}
            height=${2}
            rx=${1}
            stroke="none"
            fill=${data.faceColor}
          />
          <rect
            x=${20 + data.eyeSpread}
            y=${14}
            width=${1.5}
            height=${2}
            rx=${1}
            stroke="none"
            fill=${data.faceColor}
          />
        </g>
      </g>
    </svg>
  `;
};


function generateData(name: string, colors: string[]): {
  wrapperColor: string;
  faceColor: string;
  backgroundColor: string;
  wrapperTranslateX: number;
  wrapperTranslateY: number;
  wrapperRotate: number;
  wrapperScale: number;
  isMouthOpen: boolean;
  isCircle: boolean;
  eyeSpread: number;
  mouthSpread: number;
  faceRotate: number;
  faceTranslateX: number;
  faceTranslateY: number;
} {
  const numFromName = hashCode(name);
  const range = colors.length;
  const wrapperColor = getRandomColor(numFromName, colors, range);
  const preTranslateX = getUnit(numFromName, 10, 1);
  const wrapperTranslateX = preTranslateX < 5 ? preTranslateX + SIZE / 9 : preTranslateX;
  const preTranslateY = getUnit(numFromName, 10, 2);
  const wrapperTranslateY = preTranslateY < 5 ? preTranslateY + SIZE / 9 : preTranslateY;

  return {
    wrapperColor,
    faceColor: getContrast(wrapperColor),
    backgroundColor: getRandomColor(numFromName + 13, colors, range),
    wrapperTranslateX,
    wrapperTranslateY,
    wrapperRotate: getUnit(numFromName, 360),
    wrapperScale: 1 + getUnit(numFromName, SIZE / 12) / 10,
    isMouthOpen: getBoolean(numFromName, 2),
    isCircle: getBoolean(numFromName, 1),
    eyeSpread: getUnit(numFromName, 5),
    mouthSpread: getUnit(numFromName, 3),
    faceRotate: getUnit(numFromName, 10, 3),
    faceTranslateX: wrapperTranslateX > SIZE / 6 ? wrapperTranslateX / 2 : getUnit(numFromName, 8, 1),
    faceTranslateY: wrapperTranslateY > SIZE / 6 ? wrapperTranslateY / 2 : getUnit(numFromName, 7, 2)
  };
}


const hashCode = (name: string): number => {
  let hash = 0;
  for (let i = 0; i < name.length; i++) {
    const character = name.charCodeAt(i);
    hash = (hash << 5) - hash + character;
    hash &= hash; // Convert to 32bit integer
  }
  return Math.abs(hash);
};

const getDigit = (number: number, ntn: number): number => Math.floor(number / 10 ** ntn % 10);

const getBoolean = (number: number, ntn: number): boolean => !(getDigit(number, ntn) % 2);

const getUnit = (number: number, range: number, index?: number): number => {
  const value = number % range;

  if (index && getDigit(number, index) % 2 === 0) {
    return -value;
  }
  return value;
};

const getRandomColor = (number: number, colors: string[], range: number): string => colors[number % range] ?? '#000000';

const getContrast = (hexcolor: string): string => {

  // If a leading # is provided, remove it
  let color = hexcolor;
  if (color.startsWith('#')) {
    color = hexcolor.slice(1);
  }

  // Convert to RGB value
  const r = parseInt(color.substr(0, 2), 16);
  const g = parseInt(color.substr(2, 2), 16);
  const b = parseInt(color.substr(4, 2), 16);

  // Get YIQ ratio
  const yiq = (r * 299 + g * 587 + b * 114) / 1000;

  // Check contrast
  return yiq >= 128 ? '#000000' : '#FFFFFF';
};
```
:::


## まとめ
カッコいい感じの幾何学模様の Identicon に加えて、顔表示の Identicon。
いろいろと使い分けができそうです。

### 関連する記事
https://zenn.dev/lulzneko/articles/initial-icon-when-user-registered-github-or-other

## 参考サイト
- [boringdesigners/boring-avatars](https://github.com/boringdesigners/boring-avatars)
- [Avatar generator playground - Boring Avatars](https://boringavatars.com/)
