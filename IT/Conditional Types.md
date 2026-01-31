#IT #型 #知識 #typescript 

## Conditional Typesとは

TypeScriptの型の一つで条件付き型、型の条件分岐、条件型などと呼ばれる。
三項演算子よのうに`?`と`:`を使って `T extends U ? X : Y`のように書く。
これは`T`が`U`に割り当てが可能な場合`X`になって、そうでない場合`Y`になる。

下記のように使用することができる。

```ts
type IsString<T> = T extends string ? true : false

// const aの型はtrueとなる
const a: IsString<"a"> = true

// const b の型はfalseとなる
const b: IsString<1> = false  
```


あるobject方のプロパティを読み取り専用にする`Readonly<T>`というユーティリティ型があり、
`Readonly<T>`はそのオブジェクトの直下のプロパティを読み取り専用にしますが、
ネストしたオブジェクトのプロパティは読み取り専用にしません。
下記のようなオブジェクトがあったとします。

```ts
type Person = {
name: string;
age: number;
address: {
  country: string;
  city: string;
 };
};
```

この時`Readonly<Person>`は`address`プロパティ自体は読み取り専用の為
書き換えることができないが、
`address`の中のプロパティ`country`と`city`は書き換え可能になってしまいます。


この問題を解決する際には`Readonly<T>`を再帰的に適用する必要があります。
このような場合に[[Mapped Types]]と**Conditional Types** を組み合わせて使う。

```ts
type Freeze<T> = Readonly<{
  [P in keyof T]: T[P] extends object ? Freeze<T[P]> : T[P];
}>;
```

上記の`Freeze`関数を使用します。

```ts
const kimberley: Freeze<Person> = {
name: "Kimberley",
age: 24,
address: {
 country: "Canada",
 city: "Vancouver",
},
};


kimberley.name = "Kim";
// Cannot assign to 'name' because it is a read-only property.

kimberley.age = 25;
// Cannot assign to 'age' because it is a read-only property.

kimberley.address = {
// Cannot assign to 'address' because it is a read-only property.
country: "United States",
city: "Seattle",
};

kimberley.address.country = "United States";
//Cannot assign to 'country' because it is a read-only property.

kimberley.address.city = "Seattle";
//Cannot assign to 'city' because it is a read-only property.

```

これでネストされているプロパティ要素も書き換え不可となっているのがわかります。
`Freeze<T>`が再帰的なっているからです。