#IT #型 #知識 #typescript 

## Mapped Typesとは

[[インデックス型]]ではキーを自由に設定できてしまい、
アクセス時は毎回`undefined`かどうかの型チェックが必要になります。
入力の形式が決まっているのであれば`Mapped Types`を使用しましょう

`Mapped Types`は主に[[ユニオン型]]として組み合わせて使います。

```ts
type LangageType = "en" | "fr" | "it" | "ja"
```

これを[[インデックス型]]と同じようにキーの制約として使用することができる。

```ts
type Butterfly = {
 [key in LangageType]: string;
};
```

このように`Butterfly`を定義するとインデックス型で定義した型以外のkeyを入力すると
エラーが発生するようになります。

```ts
const butterflies: Butterfly = {
en: "Butterfly",
fr: "Papillon",
it: "Farfalla",
ja: "蝶々",
de: "Schmetterling",
//オブジェクト リテラルは既知のプロパティのみ指定できます。'de' は型 'Butterfly' に存在しません。

};
```

## Mapped Typesを使用したユーティリティ型の紹介

プロパティを読み取り専用にする`readonly`をそのオブジェクトすべての
プロパティに適用する`Readonly<T>` というユーティリティ型があります。

`Readonly<T>`もこの機能で実現されています。
```ts
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

`keyof T`はオブジェクトのkeyをユニオン方に変更するものだと解釈してください。

例：

```ts
type A = { x: number; y: string };

type B = {
  [P in keyof A]: A[P];
};
// Bの型は {x: number, y:string}となる
```

#### mapping modifier

`-`を先頭につけ`-readonly`とすることで逆に読み取り専用となっているプロパティを
変更にすることもできます

```ts
type Readonly<T> = {
  -readonly [P in keyof T]: T[P];
};
```

## インデックスアクセスの注意点

`{ [ K in string]: ... }` のようにキーを`string`など、リテラル型でない型を指定した
場合は、インデックスアクセスに注意する。
存在しないキーにアクセスしても、キーがあるかのようにあつかわれてしまう。


```ts
const dict: { [ K in string]: number } = { a: 1};
dict.b;
// この時bはundefinedのはずなのにnumber型と出てしまう。 
```

実行時にエラーが起きてしまう

```ts
const dict: { [K in string]: number } = { a: 1 };
console.log(dict.b);
//⇛undefined
dict.b.toFixed(); // 実行時エラーが発生する
```

この問題を対処するため、コンパイラオプション[[noUncheckedIndexedAccess]]があります
これを有効にすると、インデックスアクセスの結果の形が`T | undefined`になります。

```ts
const dict: { [K in string]: number } = { a: 1 };
dict.b;
//型の結果は number | undefined

dict.b.toFixed();
// 'dict.b' は'undefined'のため使用できません。
```

#### Mapped Typesには追加のプロパティが書けない

[[インデックス型]]との異なる点として追加のプロパティが書けません

```ts
type KeyValuesAndName = {
  [K in string]: string;
  name: string; // 追加のプロパティ
  //error: マップされた型では、プロパティまたはメソッドを宣言しない場合があります。
}
 ```

追加のプロパティがある場合はその部分をオブジェクトの型として定義し、
Mapped Typesと[[インターセクション型]]を成す必要があります。

```ts
type KeyValues = {
  [K in string]: string;
};

type Name = {
  name: string;
};


type KeyValuesAndName = KeyValues & Name;
```

上の例を一つの型にまとめることができます。

```ts
type KeyValuesAndName = {
  [K in string]: string
} & {
  name: string; //追加のプロパティ
} 
```