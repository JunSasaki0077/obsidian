#IT 

## 学んだこと


ログイン状態では見えて、ログアウト状態では見えないようにするには
[[三項演算子]]を使用して見えなくさせる

まずはSessionとして現在のユーザーがログインしているかどうかを検証する。

そのセッションを用いて三項演算子を使用する
```tsx
{session?.user && (ここに表示したい項目を記入)}
```

ログインしていない状態の場合見せたくない項目は
このsession?.user の中に記入する。

## 検索方法

[[nuqs]]というライブラリを使用する。
検索した際に１文字１文字でレンダリングが発生してしまうため
オプションのデバウンスというのを使用し、
入力後0.5秒後にレンダリングするなど制御できる。****


## Page Props

[[PageProps]]とは

## map関数petsの型エラー解消法

[[searcParams]]を用いて検索機能を実装している際に
`petCard`をmap関数で読み込んでいる場所で
`petsはundefinedの可能性があります`とエラーを吐いた

```tsx
import PetCard from '@/components/pet-card';
import PetSearchForm from '@/components/pet-search-form';
import { getPets, searchPets } from '@/data/pet';
import { createLoader, parseAsString } from 'nuqs/server';

const loadSearchParams = createLoader({
name: parseAsString.withDefault(''),
});

const PetsPage = async ({ searchParams }: PageProps<'/pets'>) => {
const { name } = await loadSearchParams(searchParams);
const pets = name ? await searchPets(name) : await getPets();

return (
<div className='container py-10'>
<PetSearchForm />
<h1 className='text-2xl font-bold mb-6'>ペット一覧</h1>
<div className='grid grid-cols-3 gap-4'>
{pets.map((pet) => (
<div key={pet.id}>
<PetCard pet={pet} />
</div>
))}
</div>
</div>
);
};
export default PetsPage;
```

petsの名前があった場合は検索結果のペットを表示し、
ない場合はデータベースから取得したペットを表示するといった流れ

今までのpetsの型は

```ts
const pets: {  
type: "dog" | "cat";  
name: string;  
id: string;  
hp: number;  
ownerId: string;  
}
```
ひとつの配列として型を取れていたのに
これを追加した際に今まで大丈夫だったmap関数の
petsの形が変化してしまった

```tsx
const { name } = await loadSearchParams(searchParams);
const pets = name ? await searchPets(name) : await getPets();
```

```ts
const pets: {  
type: "dog" | "cat";  
name: string;  
id: string;  
hp: number;  
ownerId: string;  
} | {  
type: "dog" | "cat";  
name: string;  
id: string;  
hp: number;  
ownerId: string;  
}[] | undefined
```


#### 検証と解消法

そもそもundefinedじゃないようにすればいいと考え
[[mapメソッド]]のpetsに`!`の[[ノンアサーション演算子]]や`?`の[[オプショナル]] を
使用したが結局

```ts
プロパティ 'map' は型 '{ type: "dog" | "cat"; name: string; id: string; hp: number; ownerId: string; } | { type: "dog" | "cat"; name: string; id: string; hp: number; ownerId: string; }[]' に存在しません
```

とundefinedではないけど、`オブジェクト`か`配列`ではないよねと怒られてしまう。
そもそも[[mapメソッド]]では配列以外はエラーを吐く


## favicon

apple-touch-iconをapple-iconに変更することにより
iphoneでホーム画面の画像を指定することができる。

androidの場合は１番大きいサイズのicon.pngにすることで指定できる。


## Seed

drizze-seedを用いることで