
2025/01/08

#TailwindCSS #IT

## はじめに

tailwindCSS の v4.0 ベータ版が出ました！
それに伴って今まで以上に便利な機能が追加されたので
細かい変更やよく使う機能を絞って紹介させていただきます！

## 初期設定

初期設定の部分から変更されています。
まず CSS ファイルに必要な記述が少なくなりました！

```diff css:app.css
- @tailwind base;
- @tailwind components;
- @tailwind utilities;

+ @import "tailwindcss";
```

見ての通り今までは TailWindCSS を使うのに
必要なコードが 3 行ありましたが
1 行になりました。

### v3 からのアップデート

今現在使用している v3 からアップデートする際は

```
$ npx @tailwindcss/upgrade@next
```

この一文ターミナルにペッとするだけで
自動で v4 にアップデートしてくれます。

> アップデートを行うには
> Node.js20 以上が必要となります。

## テーマ変更

TailWindCSS にある初期コードを
@theme とすることで
変更、追加を行うことができます。

```css: app.css
@theme {
 --text-2xs: 10px;
 --text-xs: 11px;
}
```

```tsx:index.tsx
<div className="text-2xs">10pxになります</div>
<div className="text-xs">11pxになります</div>
```

上記のコードは本来 TailWindCSS では
text-2xs はないのですが、10px として追加し、
text-xs のピクセルは本来 12px なのですが
11px に変更を行っております。

また、

```css:app.css
@theme {
--font-*: initial;
--*: initial;
}
```

上記のコードは
`--○○-*:initialt;` とすることで
〇〇に入るプロパティのテーマを
削除することができます。
`--*: initial;`は
TailWindCSS にあるデフォルトの
テーマを全て解除することができます。

## カスケードレイヤード

@layer と行うことで 独自の layer の作成や
スタイルの優先順位とスタイル同士の
相互作用を制御することができるようになりました。

従来の CSS のカスケードレイヤードを採用しているので
詳しくは下記のリンクをご覧ください。
[MDN @layer リンク](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer)

## 自動ソース検出

tailwind.config 等で@app のように
ファイルを指定していたと思いますが,
v4 では自動でソースから検出するようになった為、
設定が不要になりました。

また＠source ~~ とファイルを指定することにより
TailWindCSS ファイルを個別で使用することができます。

## arbitary values の簡素化

そもそも arbitary values とは
[]の間に任意の値を指定することで
class があらかじめ用意されていない
CSS のプロパティ値を利用することが
できる機能のことです。

こちらの機能が全てのプロパティに
ではないですが簡素化されるようになりました。

```diff tsx:index.tsx
- <div className="z-[9999]"></div>
+ <div className="z-9999"></div>
```

## data 属性の省略化

上記と同じく data 属性も省略化することが
できるようになりました。

```diff tsx:index.tsx
- <div class="opacity-50 data-[selected]:opacity-100" data-selected>
+ <div class="opacity-50 data-selected:opacity-100" data-selected>
```

## スケール のデフォルト値の変更

v3 の TailWindCSS では
width や margin 等を指定した際
1 ごとに 0.25rem 付与されていましたが、
v4 では 1 ごとに`calc(var(--spacing) * 1)`
が付与されるようになりました。
--spacing のデフォルト値は 0.25rem ですが
こちらを上記でも紹介しましたが
`@theme`を用いることで
デフォルトの値を変更することができるようになりました。

```diff css:app.css

@theme {
- --spacing: 0.25rem;
+ --spacing: 0.5rem
}
```

## コンテナクエリのサポート

v3 ではコンテナクエリを使用する際に
`@tailwindcss/container-queries`プラグインが必要でしたが
v4 ではその必要がなくなりました。

そもそもコンテナクエリとはですが

```html

<div class="@container">
  <div class="grid grid-cols-1 @sm:grid-cols-3 @lg:grid-cols-4">
    <!-- ... -->
  </div>
</div>

```

親要素に`@container`として
子要素に`@sm,@md`などを指定することによって
親要素の幅によって CSS を付与することができるようになります。

## テキストエリアの高さ調整

textarea に`field-sizing-content`を指定することで
フィールド内の要素に応じて自動的にサイズを変更してくれます。
今までであれば auto-text-size 等のライブラリを使用して
いましたが、それが不要になりました。便利ですね。

## バリアントの連携

```diff html
<div class="group">
-  <div class="group-has-[&[data-potato]]:opacity-100">
+  <div class="group-has-data-potato:opacity-100">
    <!-- ... -->
  </div>
  <div data-potato>
    <!-- ... -->
  </div>
</div>
```

v3 では他のバリアントと連携させるには
複雑なコードが必要でしたが、
v4 ではシンプルにコードを書けるようになりました。

## not-\* バリアント

新しく not-\*バリアントが追加されました。
先ほどのコードで紹介すると
`<div class="group-not-has-data-potato:opacity-100">`
とすることで group 内に data-potato がない場合
opacity-100 を適用する css となります。

## nth-\* バリアント

nth-child の擬似クラスに 4 つの新しいバリアントが追加されました。

```tsx:index.tsx
<div class="nth-3:underline">…</div>
<div class="nth-last-5:underline">…</div>
<div class="nth-of-type-4:underline">…</div>
<div class="nth-last-of-type-6:underline">…</div>
```

## Descendant バリアント

子孫要素全てに CSS を付与する`**`バリアントが追加されました。

```tsx:index.tsx
<div class="**:data-avatar:rounded-full">
  <div>
    <img src="…" data-avatar />
  </div>
  <p>…</p>
</div>
```

上記 data-avatar がある
img タグに rounded-full が適用されます。

ちなみに`*`とすることで子要素に CSS を
付与することができます。

## プラグインの使用

従来の TailWindCSS ではプラグイン使用時は npm
でインストールしてたと思いますが
v4 では`@plugin`ディレクティブで
使用することができます。

```css:app.css
@import "tailwindcss";
@plugin "@tailwindcss/typography";
```

## JS ファイルの設定

v3 から一気に v4 にアップデートしたいけど
段階的にアップデートしたい場合など
設定ファイルだけは JS 使いたい場合などに
`@config`とすることで使用することができます。

```css:app.css
@import "tailwindcss";
@config "YOUR_PATH/tailwind.config.js";
```

## まとめ

最後まで読んでいただきありがとうございました。
さっくりではありますが v4 の
機能紹介させていただきました。
他にも色々な機能が公式ドキュメントには
あるのでぜひ見てみてください。

[公式ドキュメント](https://tailwindcss.com/docs/v4-beta#using-the-theme-function)
