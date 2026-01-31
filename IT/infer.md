#知識 #IT #typescript 

## inferとは

[[Conditional Types]]の中で使われる型演算子で`infer`は「推論する」という意味です。
`extends`の右辺にのみ書くことができる。

#### ユーティリティ型`ReturnType<T>`の例から`infer`を知る

ある関数の戻り値の方を取得するユーティリティ型`ReturnType<T>`があり、
次のように定義されています。

```ts
type ReturnType<T extends (...args: any) => any> T extends (
  ...args: any
) => infer R ? R :any
```

使用した例：

```ts
const request = (url: string): Promise<string> => {
  return fetch(url).then((res) => res.text());
};

type X = ReturnType<typeof request>;
// Xのタイプは Promise<string>
```

[[typeof]]は変数から型を取得する演算子です。JavaScriptの`typeof`とは異なるので注意が必要です。

このように関数`request`の型から戻り値の型を取得することができました。

#### `ReturnType<T>`の解説

`ReturnType<T>`の構造を知るためにはまず`T extends (...args: any) => any`が
何かを知る必要があります。
これは一般的な関数の型を示しています。任意のこすうでにんいの方の引数を受け取り、
任意の型の値を返すことを示しています。`T`は任意の関数を示しています。
戻り値が`=> infer R ? R : any`となっており、`T`が関数である場合は戻り値の型である
`R`、そうでない場合は`any`を返すという意味になっております。

`infer`を使うことによってある型`T`が廃鉄ふぇある場合はその要素の型、
そうでない場合は`never`を返す`Flatten<T>`を作成してみましょう。