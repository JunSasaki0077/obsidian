#IT 

## 学んだこと


ログイン状態では見えて、ログアウト状態では見えないようにするには
三項演算子を使用して見えなくさせる

まずはSessionとして現在のユーザーがログインしているかどうかを検証する。

そのセッションを用いて三項演算子を使用する
```tsx
{session?.user && (ここに表示したい項目を記入)}
```

ログインしていない状態の場合見せたくない項目は
このsession?.user の中に記入する。

## 検索方法

検索した際に１文字１文字でレンダリングが発生してしまうため
デバウンスというのを使用し、
入力後0.5秒後にレンダリングするなど制御できる。


## Page Props

[[PageProps]]とは

## Petsのエラー解消法

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

これを追加した際に起きた、このコードは
petsの名前があった場合は検索結果のペットを表示し、
ない場合はデータベースから取得したペットを表示するといったながれだ