
title: "はじめに"
---

Next.jsには2つのRouterが同梱されています。

- [App Router↗︎](https://nextjs.org/docs/app): 現在主流の新しいRouter
- [Pages Router↗︎](https://nextjs.org/docs/pages): 従来のRouter

App Routerは[React Server Components↗︎](https://ja.react.dev/learn/creating-a-react-app#which-features-make-up-the-react-teams-full-stack-architecture-vision)（**RSC**）はじめReactの先進的な機能をサポートするフレームワークであり、Pages Routerとは機能・設計・プラクティスなどあらゆる面で大きく異なります。

本書は、Next.jsやRSCの根底にある**考え方**に基づいた設計やプラクティスをまとめたものです。公式ドキュメントはじめ、筆者や筆者の周りで共有されている理解・前提知識などを元にまとめています。

本書を通じて、Next.jsに対する読者の理解を一層深めることができれば幸いです。

:::message alert

- 本書はNext.js v15系を前提としています。
- 本書では断りがない限り「Next.js」はApp Routerを指します。

:::

## 対象読者

対象読者は以下を想定してます。

- Next.jsを説明できるが深く使ったことはない初学者
- Next.jsを用いた開発で苦戦している中級者

初学者にもわかりやすい説明になるよう心がけましたが、本書は**入門書ではありません**。そのため、前提知識として説明を省略している部分もあります。入門書としては[公式のLearn↗︎](https://nextjs.org/learn)や[実践Next.js↗︎](https://gihyo.jp/book/2024/978-4-297-14061-8)などをお勧めします。

https://nextjs.org/learn

https://gihyo.jp/book/2024/978-4-297-14061-8

## 謝辞

本書の執筆にあたり、[koichikさん↗︎](https://x.com/koichik)にレビュー協力をいただきました。多大な時間を割いてレビューや議論にお付き合いいただいたおかげで、本書をより良いものにできました。本当にありがとうございます。

## 変更履歴

- 2025/10: [クライアントとサーバーのバンドル境界](part_2_bundle_boundary)追加、全体の改定<!-- https://github.com/AkifumiSato/zenn-article/pull/81 -->
- 2025/01: Next.js v15対応<!-- https://github.com/AkifumiSato/zenn-article/pull/69 -->
- 2024/10: [Server Componentsの純粋性](part_4_pure_server_components)や[第5部 その他のプラクティス](part_5)の追加、一部校正<!-- https://github.com/AkifumiSato/zenn-article/pull/67 -->
- 2024/08: 初稿<!-- https://github.com/AkifumiSato/zenn-article/pull/65 -->

---
title: "第1部 データフェッチ"
---

Next.jsがサポートする[RSC↗︎](https://ja.react.dev/reference/rsc/server-components)では、従来のReactフレームワークから多くのパラダイムシフトを必要とします。データフェッチにおいては、Server Componentsによってセキュアでシンプルな実装が可能になった一方、従来とは全く異なる設計思想が求められます。

第1部では、Next.jsのデータフェッチにまつわる基本的な考え方を解説します。

:::message
Next.jsにおけるデータアクセスのアプローチは、バックエンドAPIを分離して開発するか、Next.jsにデータベースアクセスを統合するかの2つに分けられます。本書で扱う「考え方」がこれらの選択によって大きく変わる部分は少ないと思われることもあり、本書では**バックエンドAPIを分離**するアプローチを前提に解説します。
:::

---
title: "データフェッチ on Server Components"
---

## 要約

データフェッチはClient Componentsではなく、Server Componentsで行いましょう。

## 背景

Reactにおけるコンポーネントは従来クライアントサイドでの処理を主体としていたため、クライアントサイドにおけるデータフェッチのためのライブラリや実装パターンが多く存在します。

- [SWR↗︎](https://swr.vercel.app/)
- [React Query↗︎](https://react-query.tanstack.com/)
- GraphQL
  - [Apollo Client↗︎](https://www.apollographql.com/docs/react/)
  - [Relay↗︎](https://relay.dev/)
- [tRPC↗︎](https://trpc.io/)
- etc...

しかしクライアントサイドでデータフェッチを行うことは、多くの点でデメリットを伴います。

### パフォーマンスと設計のトレードオフ

クライアント・サーバー間の通信は、物理的距離や不安定なネットワーク環境の影響で多くの場合低速です。そのため、パフォーマンス観点では通信回数が少ないことが望ましいですが、通信回数とシンプルな設計はトレードオフになりがちです。

REST APIにおいて通信回数を優先すると**God API**と呼ばれる責務が大きなAPIになりがちで、変更容易性やAPI自体のパフォーマンス問題が起きやすい傾向にあります。一方責務が小さい細粒度なAPIは**Chatty API**(おしゃべりなAPI)と呼ばれ、データフェッチをコロケーション^[コードをできるだけ関連性のある場所に配置することを指します。]してカプセル化などのメリットを得られる一方、通信回数が増えたりデータフェッチのウォーターフォールが発生しやすく、Webアプリのパフォーマンス劣化要因になりえます。

### 様々な実装コスト

クライアントサイドのデータフェッチでは多くの場合、[Reactが推奨↗︎](https://ja.react.dev/reference/react/useEffect#what-are-good-alternatives-to-data-fetching-in-effects)してるようにキャッシュ機能を搭載した3rd partyライブラリを利用します。一方リクエスト先に当たるAPIは、パブリックなネットワークに公開するためより堅牢なセキュリティが求められます。

これらの理由からクライアントサイドでデータフェッチする場合には、3rd partyライブラリの学習・責務設計・API側のセキュリティ対策など様々な開発コストが発生します。

### バンドルサイズの増加

クライアントサイドでデータフェッチを行うには、3rd partyライブラリ・データフェッチの実装・バリデーションなど、多岐にわたるコードがバンドルされクライアントへ送信されます。また、通信結果次第では利用されないエラー時のUIなどのコードもバンドルに含まれがちです。

## 設計・プラクティス

Reactチームは前述の問題を個別の問題と捉えず、根本的にはReactがサーバーをうまく活用できてないことが問題であると捉えて解決を目指しました。その結果生まれたのが**React Server Components**（**RSC**）アーキテクチャです。

https://ja.react.dev/reference/rsc/server-components

Next.jsはRSCをサポートしており、[データフェッチはServer Componentsで行う↗︎](https://nextjs.org/docs/app/getting-started/fetching-data)ことがベストプラクティスとされています。

:::message alert
「Server Componentsには`"use server"`が必要」という誤解が散見されますが、これは**誤り**です。`"use server"`は関数を[Server Functions↗︎](https://ja.react.dev/reference/rsc/server-functions)としてマークし、クライアントサイドから呼び出し可能にするものであり、Server Componentsに指定するためのものではありません。

詳しくは[クライアントとサーバーのバンドル境界](part_2_bundle_boundary)を参照ください。
:::

データフェッチをServer Componentsで行うにより、以下のようなメリットを得られます。

### 高速なバックエンドアクセス

Next.jsサーバーとAPIサーバー間の通信は、多くの場合高速で安定しています。特に、APIが同一ネットワーク内や同一データセンターに存在する場合は非常に高速です。APIサーバーが外部にある場合も、多くの場合は首都圏内で高速なネットワーク回線を通じての通信になるため、比較的高速で安定してることが多いと考えられます。

### シンプルでセキュアな実装

Server Componentsは非同期関数をサポートしており、3rd partyライブラリなしでデータフェッチをシンプルに実装できます。

```tsx
export async function ProductTitle({ id }) {
  const res = await fetch(`https://dummyjson.com/products/${id}`);
  const product = await res.json();

  return <div>{product.title}</div>;
}
```

これはServer Componentsがサーバー側で1度だけレンダリングされ、従来のようにクライアントサイドで何度もレンダリングされることを想定しなくて良いからこそできる設計です。

また、データフェッチはサーバー側でのみ実行されるため、APIをパブリックなネットワークで公開することは必須ではありません。プライベートなネットワーク内でのみバックエンドAPIへアクセスするようにすれば、セキュリティリスクや対策コストを軽減できます。

### バンドルサイズの軽減

Server Componentsの実行結果はHTMLやRSC Payloadとしてクライアントへ送信されます。そのため、前述のような

> 3rd partyライブラリ・データフェッチの実装・バリデーションなど、多岐にわたるコード
> ...
> エラー時のUIなどのコード

は一切バンドルには含まれません。

## トレードオフ

### ユーザー操作とデータフェッチ

ユーザー操作に基づくデータフェッチはServer Componentsで行うことが困難な場合があります。詳細は後述の[ユーザー操作とデータフェッチ](part_1_interactive_fetch)を参照してください。

### GraphQLとの相性の悪さ

RSCにGraphQLを組み合わせることは**メリットよりデメリットの方が多くなる**可能性があります。

GraphQLはその特性上、前述のようなパフォーマンスと設計のトレードオフが発生しませんが、RSCも同様にこの問題を解消するため、これをメリットとして享受できません。それどころか、RSCとGraphQLを協調させるための知見やライブラリが一般に不足してるため、実装コストが高くバンドルサイズも増加するなど、デメリットが多々含まれます。

:::message
RSCの最初のRFCは、Relayの初期開発者の1人でGraphQLを通じてReactにおけるデータフェッチのベストプラクティスを追求してきた[Joe Savona氏↗︎](https://twitter.com/en_js)によって提案されました。そのため、RSCはGraphQLの持っているメリットや課題を踏まえて設計されているという**GraphQLの精神的後継**の側面を持ち合わせていると考えることができます。

より詳しくは、[Reactチームが見てる世界、Reactユーザーが見てる世界↗︎](https://zenn.dev/akfm/articles/react-team-vision)で解説しているので、ご参照ください。
:::

---
title: "データフェッチ コロケーション"
---

## 要約

データフェッチはデータを参照するコンポーネントにコロケーション^[コードをできるだけ関連性のある場所に配置することを指します。]し、コンポーネントの独立性を高めましょう。

<!-- 参考 https://kentcdodds.com/blog/colocation -->

## 背景

Pages Routerにおけるサーバーサイドでのデータフェッチは、[getServerSideProps↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)や[getStaticProps↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)などページコンポーネントの外側で非同期関数を宣言し、Next.jsが実行結果をpropsとしてページコンポーネントに渡すという設計がなされてました。

これはいわゆる**バケツリレー**(Props Drilling)と呼ばれるpropsを親から子・孫へと渡していくような実装を必要とし、冗長で依存関係が広がりやすいというデメリットがありました。

### 実装例

以下は商品ページを想定した実装例です。APIから取得した`product`というpropsが親から孫までそのまま渡されるような実装が見受けれれます。

```tsx
type ProductProps = {
  product: Product;
};

export const getServerSideProps = (async () => {
  const res = await fetch("https://dummyjson.com/products/1");
  const product = await res.json();
  return { props: { product } };
}) satisfies GetServerSideProps<ProductProps>;

export default function ProductPage({
  product,
}: InferGetServerSidePropsType<typeof getServerSideProps>) {
  return (
    <ProductLayout>
      <ProductContents product={product} />
    </ProductLayout>
  );
}

function ProductContents({ product }: ProductProps) {
  return (
    <>
      <ProductHeader product={product} />
      <ProductDetail product={product} />
      <ProductFooter product={product} />
    </>
  );
}

// ...
```

わかりやすいよう少々大袈裟に実装していますが、こういったバケツリレー実装はPages Routerだと発生しがちな問題です。常に最上位で必要なデータを意識し末端まで流すので、コンポーネントのネストが深くなるほどバケツリレーは増えていきます。

この設計は我々開発者に常にページという単位を意識させてしまうため、コンポーネント指向な開発と親和性が低く、高い認知負荷を伴います。

## 設計・プラクティス

App RouterではServer Componentsが利用可能なので、末端のコンポーネントへ**データフェッチをコロケーション**することを推奨^[公式ドキュメントにおける[ベストプラクティス↗︎](https://nextjs.org/docs/14/app/building-your-application/data-fetching/patterns#fetching-data-where-its-needed)を参照ください。]しています。

もちろんページの実装規模にもよるので、小規模な実装であればページコンポーネントでデータフェッチしても問題はないでしょう。しかし、ページコンポーネントが肥大化していくと中間層でのバケツリレーが発生しやすくなるので、できるだけ末端のコンポーネントでデータフェッチを行うことを推奨します。

「それでは全く同じデータフェッチが何度も実行されてしまうのではないか」と懸念される方もいるかもしれませんが、App Routerでは[Request Memoization↗︎](https://nextjs.org/docs/app/guides/caching#request-memoization)によってデータフェッチがメモ化されるため、全く同じデータフェッチが複数回実行されることないように設計されています。

### 実装例

前述の商品ページの実装例をApp Routerに移行する場合、以下のような実装になるでしょう。

```tsx
type ProductProps = {
  product: Product;
};

// <ProductLayout>は`layout.tsx`へ移動
export default function ProductPage() {
  return (
    <>
      <ProductHeader />
      <ProductDetail />
      <ProductFooter />
    </>
  );
}

async function ProductHeader() {
  const res = await fetchProduct();

  return <>...</>;
}

async function ProductDetail() {
  const res = await fetchProduct();

  return <>...</>;
}

// ...

async function fetchProduct() {
  // Request Memoizationにより、実際のデータフェッチは1回しか実行されない
  const res = await fetch("https://dummyjson.com/products/1");
  return res.json();
}
```

データフェッチが各コンポーネントにコロケーションされたことで、バケツリレーがなくなりました。また、`<ProductHeader>`や`<ProductDetail>`などの子コンポーネントはそれぞれ必要な情報を自身で取得しているため、ページ全体でどんなデータフェッチを行っているか気にする必要がなくなりました。

## トレードオフ

### Request Memoizationへの理解

データフェッチのコロケーションを実現する要はRequest Memoizationなので、Request Memoizationに対する理解と最適な設計が重要になってきます。

この点については次の[Request Memoization](part_1_request_memoization)の章でより詳細に解説します。

---
title: "Request Memoization"
---

## 要約

データフェッチ層を分離して、Request Memoizationを生かせる設計を心がけましょう。

## 背景

[データフェッチ コロケーション](part_1_colocation)の章で述べた通り、Next.jsではデータフェッチをコロケーションすることが推奨されています。末端のコンポーネントでデータフェッチを行うことはリクエストの重複リスクを伴いますが、Next.jsでは[**Request Memoization**↗︎](https://nextjs.org/docs/app/guides/caching#request-memoization)（リクエストのメモ化）によってレンダリング中の同一リクエストを排除します。

しかし、Next.jsがリクエストを重複と判定するには、同一URL・同一オプションの指定が必要で、オプションが1つでも異なれば別リクエストが発生してしまいます。

## 設計・プラクティス

オプションの指定ミスによりRequest Memoizationが効かないことなどがないよう、複数のコンポーネントで利用しうるデータフェッチ処理は**データフェッチ層**として分離しましょう。

```ts
// プロダクト情報取得のデータフェッチ層
export async function getProduct(id: string) {
  const res = await fetch(`https://dummyjson.com/products/${id}`, {
    // 独自ヘッダーなど
  });
  return res.json();
}
```

### ファイル構成

Next.jsではコロケーションを強く意識した設計がなされているので、データフェッチ層をファイル分離する場合にもファイルコロケーションすることが推奨されます。

前述の`getProduct()`を分離する場合、筆者なら以下のいずれかのような形でファイルを分離します。データフェッチ層が多い場合にはより細かく分離すると良いでしょう。

| ファイル                               | 補足                                                                |
| -------------------------------------- | ------------------------------------------------------------------- |
| `app/products/fetcher.ts`              | `products/`以下にファイルが少ない<br>データフェッチ関数も少ない場合 |
| `app/products/_lib/fetcher.ts`         | `products/`以下にファイルが多い<br>データフェッチ関数は少ない場合   |
| `app/products/_lib/fetcher/product.ts` | データフェッチ関数が多い場合                                        |

ファイルの命名やディレクトリについては開発規模や流儀によって異なるので、自分たちのチームでルールを決めておきましょう。

### `server-only`

[データフェッチ on Server Components](part_1_server_components)で述べたとおり、データフェッチは基本的にServer Componentsで行うことが推奨されます。データフェッチ層を誤ってクライアントサイドで利用することを防ぐためにも、[server-only↗︎](https://www.npmjs.com/package/server-only)を利用してモジュールを保護することを推奨します。

```ts
// Client Bundle内でimportするとエラー
import "server-only";

export async function getProduct(id: string) {
  const res = await fetch(`https://dummyjson.com/products/${id}`, {
    // 独自ヘッダーなど
  });
  return res.json();
}
```

## トレードオフ

特になし

---
title: "並行データフェッチ"
---

## 要約

以下のパターンを駆使して、データフェッチが可能な限り並行になるよう設計しましょう。

- [データフェッチ単位のコンポーネント分割](#データフェッチ単位のコンポーネント分割)
- [並行`fetch()`](#並行fetch)
- [preloadパターン](#preloadパターン)

## 背景

データフェッチ由来のデータ間に依存関係がある場合、データフェッチは直列(ウォーターフォール)に実行せざるを得ません。

一方、データ間に依存関係がない場合、データフェッチを並行化すれば優れたパフォーマンスを得られます。以下は[公式ドキュメント↗︎](https://nextjs.org/docs/14/app/building-your-application/data-fetching/patterns#parallel-and-sequential-data-fetching)にあるデータフェッチの並行化による速度改善のイメージ図です。

![water fall data fetch](/images/nextjs-basic-principle/sequential-fetching.png)

## 設計・プラクティス

Next.jsにおけるデータフェッチの並行化にはいくつかの実装パターンがあります。コードの凝集度を考えると、まずは可能な限り**データフェッチ単位のコンポーネント分割**を行うことがベストです。ただし、必ずしもコンポーネントが分割可能とは限らないので他のパターンについてもしっかり理解しておきましょう。

### データフェッチ単位のコンポーネント分割

データ間に依存関係がなく参照単位も異なる場合には、データフェッチを行うコンポーネント自体分割することを検討しましょう。

非同期コンポーネントは兄弟もしくは兄弟の子孫コンポーネントとして配置されてる場合、並行にレンダリングされます^[非同期コンポーネントがネストしている場合はコンポーネントの実行が直列になります。]。

```tsx
function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;

  return (
    <>
      <PostBody postId={id} />
      <CommentsWrapper>
        <Comments postId={id} />
      </CommentsWrapper>
    </>
  );
}

async function PostBody({ postId }: { postId: string }) {
  const res = await fetch(`https://dummyjson.com/posts/${postId}`);
  const post = (await res.json()) as Post;
  // ...
}

async function Comments({ postId }: { postId: string }) {
  const res = await fetch(`https://dummyjson.com/posts/${postId}/comments`);
  const comments = (await res.json()) as Comment[];
  // ...
}
```

上記の実装例では`<PostBody />`と`<Comments />`およびその子孫は並行レンダリングされるので、データフェッチも並行となります。

### 並行`fetch()`

データフェッチ順には依存関係がなくとも参照の単位が不可分な場合には、`Promise.all()`や`Promise.allSettled()`を利用することで、複数のデータフェッチを並行に実行できます。

```tsx
async function Page() {
  const [user, posts] = await Promise.all([
    fetch(`https://dummyjson.com/users/${id}`).then((res) => res.json()),
    fetch(`https://dummyjson.com/posts/users/${id}`).then((res) => res.json()),
  ]);

  // ...
}
```

### preloadパターン

コンポーネント構造上親子関係にせざるを得ない場合も、データフェッチにウォーターフォールが発生します。このようなウォーターフォールは、Request Memoizationを活用した[preloadパターン↗︎](https://nextjs.org/docs/app/getting-started/fetching-data#preloading-data)を利用することで、並行データフェッチを実現できます。

:::message
サーバー間通信は物理的距離・潤沢なネットワーク環境などの理由から安定して高速な傾向にあり、ウォーターフォールがパフォーマンスに及ぼす影響はクライアントサイドと比較すると小さくなる傾向にあります。それでも無視できないパフォーマンスボトルネックがある場合には、このpreloadパターンが有用です。
:::

```ts :app/fetcher.ts
import "server-only";

export const preloadCurrentUser = () => {
  // preloadなので`await`しない
  void getCurrentUser();
};

export async function getCurrentUser() {
  const res = await fetch("https://dummyjson.com/user/me");
  return res.json() as User;
}
```

```tsx :app/products/[id]/page.tsx
export default function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;

  // `<Product>`や`<Comments>`のさらに子孫で`user`を利用するため、
  // 親コンポーネントでpreloadする
  preloadCurrentUser();

  return (
    <>
      <Product productId={id} />
      <Comments productId={id} />
    </>
  );
}
```

上記実装例では可読性のためにpreload専用の関数として`preloadCurrentUser()`を定義しています。ページレベルで`preloadCurrentUser()`することで、`<Product>`と`<Comments>`のレンダリングと並行してUser情報のデータフェッチが実行されます。

ただし、preloadパターンを利用した後で`<Product>`や`<Comments>`からUser情報が参照されなくなった場合、`preloadCurrentUser()`が残っていると不要なデータフェッチが発生します。このパターンを利用する際には、無駄なpreloadが残ってしまうことのないよう注意しましょう。

## トレードオフ

### N+1データフェッチ

データフェッチ単位を小さくコンポーネントに分割していくと**N+1データフェッチ**が発生する可能性があります。この点については次の章の[N+1とDataLoader](part_1_data_loader)で詳しく解説します。

---
title: "N+1とDataLoader"
---

## 要約

コンポーネント単位の独立性を高めるとN+1データフェッチが発生しやすくなるので、DataLoaderのバッチ処理を利用して解消しましょう。

:::message
本章の内容は、バックエンドAPI側でもN+1データフェッチに対処するためのエンドポイントが実装されていることを前提としています。
:::

## 背景

前述の[データフェッチ コロケーション](part_1_colocation)や[並行データフェッチ](part_1_concurrent_fetch)を実践し、データフェッチやコンポーネントを細かく分割していくと、ページ全体で発生するデータフェッチの管理が難しくなり2つの問題を引き起こします。

1つは重複したデータフェッチです。これについてはNext.jsの機能である[Request Memoization](part_1_request_memoization)によって解消されるため、前述のようにデータフェッチ層を分離・共通化していればほとんど問題ありません。

もう1つは、いわゆる**N+1**^[参考: [[解説] SQLクエリのN+1問題↗︎](https://qiita.com/muroya2355/items/d4eecbe722a8ddb2568b)]なデータフェッチです。データフェッチを細粒度に分解していくと、N+1データフェッチになる可能性が高まります。

以下の例では投稿の一覧を取得後、子コンポーネントで著者情報を取得しています。

```tsx :page.tsx
import { type Post, getPosts, getUser } from "./fetcher";

export const dynamic = "force-dynamic";

export default async function Page() {
  const { posts } = await getPosts();

  return (
    <>
      <h1>Posts</h1>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>
            <PostItem post={post} />
          </li>
        ))}
      </ul>
    </>
  );
}

async function PostItem({ post }: { post: Post }) {
  const user = await getUser(post.userId);

  return (
    <>
      <h3>{post.title}</h3>
      <dl>
        <dt>author</dt>
        <dd>{user?.username ?? "[unknown author]"}</dd>
      </dl>
      <p>{post.body}</p>
    </>
  );
}
```

```ts :fetcher.ts
export async function getPosts() {
  const res = await fetch("https://dummyjson.com/posts");
  return (await res.json()) as {
    posts: Post[];
  };
}

type Post = {
  id: number;
  title: string;
  body: string;
  userId: number;
};

export async function getUser(id: number) {
  const res = await fetch(`https://dummyjson.com/users/${id}`);
  return (await res.json()) as User;
}

type User = {
  id: number;
  username: string;
};
```

ページレンダリング時に`getPosts()`を1回と`getUser()`をN回呼び出すことになり、ページ全体では以下のようなN+1回のデータフェッチが発生します。

- `https://dummyjson.com/posts`
- `https://dummyjson.com/users/1`
- `https://dummyjson.com/users/2`
- `https://dummyjson.com/users/3`
- ...

## 設計・プラクティス

上記のようなN+1データフェッチを避けるため、API側では`https://dummyjson.com/users/?id=1&id=2&id=3...`のように、idを複数指定してUser情報を一括で取得できるよう設計するパターンがよく知られています。

このようなバックエンドAPIと、Next.js側で[DataLoader↗︎](https://github.com/graphql/dataloader)を利用することで前述のようなN+1データフェッチを解消することができます。

### DataLoader

DataLoaderはGraphQLサーバーなどでよく利用されるライブラリで、データアクセスをバッチ処理・キャッシュする機能を提供します。具体的には以下のような流れで利用します。

1. バッチ処理する関数を定義
2. DataLoaderのインスタンスを生成
3. 短期間^[バッチングスケジュールの詳細は[公式の説明↗︎](https://github.com/graphql/dataloader?tab=readme-ov-file#batch-scheduling)を参照ください。]に`dataLoader.load(id)`を複数回呼び出すと、`id`の配列がバッチ処理に渡される
4. バッチ処理が完了すると`dataLoader.load(id)`のPromiseが解決される

以下は非常に簡単な実装例です。

```ts
async function myBatchFn(keys: readonly number[]) {
  // keysを元にデータフェッチ
  // 実際にはdummyjsonはid複数指定に未対応なのでイメージです
  const res = await fetch(
    `https://dummyjson.com/posts/?${keys.map((key) => `id=${key}`).join("&")}`,
  );
  const { posts } = (await res.json()) as { posts: Post[] };
  return posts;
}

const myLoader = new DataLoader(myBatchFn);

// 呼び出しはDataLoaderによってまとめられ、`myBatchFn([1, 2, 3])`が呼び出される
const posts = await Promise.all([
  myLoader.load(1),
  myLoader.load(2),
  myLoader.load(3),
]);
```

### Next.jsにおけるDataLoaderの利用

Server Componentsの兄弟もしくは兄弟の子孫コンポーネントは並行レンダリングされるので、それぞれで`await myLoader.load(1);`のようにしてもDataLoaderによってバッチングされます。

DataLoaderを用いて、前述の実装例の`getUser()`を書き直してみます。

```ts :fetcher.ts
import DataLoader from "dataloader";
import * as React from "react";

// ...

const getUserLoader = React.cache(
  () => new DataLoader((keys: readonly number[]) => batchGetUser(keys)),
);

export async function getUser(id: number) {
  const userLoader = getUserLoader();
  return userLoader.load(id);
}

async function batchGetUser(keys: readonly number[]) {
  // keysを元にデータフェッチ
  // 実際にはdummyjsonはid複数指定に未対応なのでイメージです
  const res = await fetch(
    `https://dummyjson.com/users/?${keys.map((key) => `id=${key}`).join("&")}`,
  );
  const { users } = (await res.json()) as { users: User[] };
  return users;
}

// ...
```

ポイントは`getUserLoader`が`React.cache()`を利用していることです。DataLoaderはキャッシュ機能があるため、ユーザーからのリクエストを跨いでインスタンスを共有してしまうと予期せぬデータ共有につながります。そのため、**ユーザーからのリクエスト単位でDataLoaderのインスタンスを生成**する必要があり、これを実現するために[React Cache↗︎](https://ja.react.dev/reference/react/cache)を利用しています。

:::message alert
Next.jsやReactのコアメンテナである[Sebastian Markbåge氏↗︎](https://bsky.app/profile/sebmarkbage.calyptus.eu)の過去ツイート（アカウントごと削除済み）によると、React Cacheがリクエスト単位で保持される仕様は将来的に保障されたものではないようです。執筆時点では他に情報がないため、上記実装を参考にする場合には必ず動作確認を行ってください。
:::

上記のように実装することで、`getUser()`のインターフェースは変えずにN+1データフェッチを解消することができます。

## トレードオフ

### Eager Loadingパターン

ここで紹介した設計パターンはいわゆる**Lazy Loading**パターンの一種です。バックエンドAPI側の実装・パフォーマンス観点からLazy Loadingが適さない場合は**Eager Loading**パターン、つまりN+1の最初の1回のリクエストで関連する必要な情報を全て取得することを検討しましょう。

Eager Loadingパターンは偏りすぎると、密結合で責務が大きすぎるいわゆる**God API**になってしまいます。これらの詳細については次章の[細粒度のREST API設計](part_1_fine_grained_api_design)で解説します。

---
title: "細粒度のREST API設計"
---

## 要約

バックエンドのAPI設計は、フロントエンドの設計にも大きく影響をもたらします。Next.jsに対するバックエンドAPIは細粒度な単位に分割されていることが望ましく、可能な限りこれを意識した設計を行いましょう。

:::message alert
このchapterの主題は「Next.jsが呼び出すバックエンドAPIの設計」です。「Next.jsの[Route Handler↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/route)として実装するAPIの設計」ではないのでご注意ください。
:::

## 背景

昨今のバックエンドAPI開発において、最もよく用いられる設計は[REST API↗︎](https://learn.microsoft.com/ja-jp/azure/architecture/best-practices/api-design)です。REST APIのリソース単位は、識別子を持つデータやオブジェクト単位でできるだけ細かく**細粒度**に分割することが基本です。しかし、細粒度のAPIは利用者システムとの通信回数が多くなってしまうため、これを避けるためにより大きな**粗粒度**単位でAPI設計することがよくあります。

### リソース単位の粒度とトレードオフ

細粒度なAPIで通信回数が多くなることはChatty API（おしゃべりなAPI）と呼ばれ、逆に粗粒度なAPIで汎用性に乏しい状態はGod API（神API）と呼ばれます。これらはそれぞれアンチパターンとされることがありますが、実際には観点次第で最適解が異なるので、一概にアンチパターンなのではなくそれぞれトレードオフが伴うと捉えるべきです。

| リソース単位の粒度 | 設計観点 | パフォーマンス<br>(低速な通信) | パフォーマンス<br>(高速な通信) |
| ------------------ | -------- | ------------------------------ | ------------------------------ |
| 細粒度             | ✅       | ❌                             | ✅                             |
| 粗粒度             | ❌       | ✅                             | ✅                             |

### ページと密結合になりがちな粗粒度単位

Pages RouterはじめApp Router登場以前のReactアプリケーションに対するバックエンドAPIでは、API設計がページと密結合になり、汎用性や保守性に乏しくなるケースが多々見られました。

クライアントサイドでデータフェッチを行う場合、クライアント・サーバー間の通信は物理的距離や不安定なネットワーク環境の影響で低速になりがちなため、通信回数はパフォーマンスに大きく影響します。そのため、ページごとに1度の通信で完結するべく粗粒度なAPI設計が採用されることがありました。

一方、Pages Routerの[`getServerSideProps()`↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)を利用したBFFとAPIのサーバー間においては、高速なネットワークを介した通信となるため、通信回数がパフォーマンスボトルネックになる可能性は低くなります。しかし、`getServerSideProps()`はページ単位で定義するため、APIにページ単位の要求が反映され、粗粒度なAPI設計になるようなケースがよくありました。

## 設計・プラクティス

App Routerにおいては、Server Componentsによってデータフェッチのコロケーションや分割が容易になりました。このため、App Routerは細粒度で設計されたREST APIと**非常に相性が良い**と言えます。

バックエンドAPIの設計観点から言っても、細粒度で設計されたREST APIの実装はシンプルに実装できることが多く、メリットとなるはずです。

### RSCとGraphQLのアナロジー

細粒度なAPI設計はRSCで初めて注目されたものではなく、従来からGraphQL BFFに対するバックエンドAPIで好まれる傾向にありました。

https://x.com/koichik/status/1825711304834953337

このように、RSCとGraphQLには共通の思想が見えます。

[データフェッチ on Server Components](part_1_server_components)でも述べたように、RSCの最初のRFCはRelayの初期開発者の1人でGraphQLを通じてReactにおけるデータフェッチのベストプラクティスを追求してきた[Joe Savona氏↗︎](https://twitter.com/en_js)が提案しており、RSCには**GraphQLの精神的後継**という側面があります。

このようなRSCとGraphQLにおける類似点については、以下のQuramyさんの記事で詳しく解説されているので、興味がある方はご参照ください。

https://quramy.medium.com/react-server-components-%E3%81%A8-graphql-%E3%81%AE%E3%82%A2%E3%83%8A%E3%83%AD%E3%82%B8%E3%83%BC-89b3f5f41a01

## トレードオフ

### バックエンドとの通信回数

前述の通り、サーバー間通信は多くの場合高速で安定しています。そのため、通信回数が多いことはデメリットになりづらいと言えますが、アプリケーション特性にもよるので実際には注意が必要です。

[並行データフェッチ](part_1_concurrent_fetch)や[N+1とDataLoader](part_1_data_loader)で述べたプラクティスや、データフェッチ単位のキャッシュである[Data Cache↗︎](https://nextjs.org/docs/app/guides/caching)を活用して、通信頻度やパフォーマンスを最適化しましょう。

### バックエンドAPI開発チームの理解

バックエンドAPIには複数の利用者がいる場合もあるため、Next.jsが細粒度のAPIの方が都合がいいからといって一存で決めれるとは限りません。バックエンド開発チームの経験則や価値観もあります。

しかし、細粒度のAPIにすることはフロントエンド開発チームにとってもバックエンド開発チームにとってもメリットが大きく、無碍にできない要素なはずです。最終的な判断がバックエンド開発チームにあるとしても、しっかりメリットやNext.js側の背景を伝え理解を得るべく努力しましょう。

---
title: "ユーザー操作とデータフェッチ"
---

## 要約

ユーザー操作に基づくデータフェッチと再レンダリングには、Server Functionsと`useActionState()`を利用しましょう。

## 背景

[データフェッチ on Server Components](part_1_server_components)で述べた通り、Next.jsにおいてデータフェッチはServer Componentsで行うことが基本形です。しかし、Server Componentsはユーザー操作に基づいてデータフェッチ・再レンダリングを行うのに適していません。Next.jsでは`router.refresh()`などでページ全体を再レンダリングすることはできますが、部分的に再レンダリングしたい場合には不適切です。

## 設計・プラクティス

Next.jsがサポートしてるRSCにおいては、[Server Functions↗︎](https://ja.react.dev/reference/rsc/server-functions)と`useActionState()`を利用することで、ユーザー操作に基づいたデータフェッチを実現できます。

### `useActionState()`

`useActionState()`は関数と初期値を渡すことで、Server Functionsを通して更新できるState管理が実現できます。

https://ja.react.dev/reference/react/useActionState

以下はユーザーの入力に基づいて商品を検索する実装例です。Server Functionsとして定義された`searchProducts()`を`useActionState()`の第一引数に渡しており、formがサブミットされるごとに`searchProducts()`が実行されます。

```ts :app/search-products.ts
"use server";

export async function searchProducts(
  _prevState: Product[],
  formData: FormData,
) {
  const query = formData.get("query") as string;
  const res = await fetch(`https://dummyjson.com/products/search?q=${query}`);
  const { products } = (await res.json()) as { products: Product[] };

  return products;
}

// ...
```

```tsx :app/form.tsx
"use client";

import { useActionState } from "react-dom";
import { searchProducts } from "./search-products";

export default function Form() {
  const [products, action] = useActionState(searchProducts, []);

  return (
    <>
      <form action={action}>
        <label htmlFor="query">
          Search Product:&nbsp;
          <input type="text" id="query" name="query" />
        </label>
        <button type="submit">Submit</button>
      </form>
      <ul>
        {products.map((product) => (
          <li key={product.id}>{product.title}</li>
        ))}
      </ul>
    </>
  );
}
```

上記実装例では検索したい文字列を入力しサブミットすると、検索ヒットした商品の名前が一覧で表示されます。

## トレードオフ

### URLシェア・リロード対応

form以外ほとんど要素がないような単純なページであれば、公式チュートリアルの実装例のように`router.replace()`によってURLを更新・ページ全体を再レンダリングするという手段があります。

https://nextjs.org/learn/dashboard-app/adding-search-and-pagination

:::details チュートリアルの実装例(簡易版)

```tsx
"use client";

import { useSearchParams, usePathname, useRouter } from "next/navigation";

export default function Search() {
  const searchParams = useSearchParams();
  const pathname = usePathname();
  const { replace } = useRouter();

  function handleSearch(term: string) {
    // MEMO: 実際にはdebounce処理が必要
    const params = new URLSearchParams(searchParams);
    if (term) {
      params.set("query", term);
    } else {
      params.delete("query");
    }
    replace(`${pathname}?${params.toString()}`);
  }

  return (
    <input
      onChange={(e) => handleSearch(e.target.value)}
      defaultValue={searchParams.get("query")?.toString()}
    />
  );
}
```

```tsx
export default async function Page(props: {
  searchParams?: Promise<{
    query?: string;
    page?: string;
  }>;
}) {
  const searchParams = await props.searchParams;
  const query = searchParams?.query || "";
  const currentPage = Number(searchParams?.page) || 1;

  return (
    <div>
      <Search />
      {/* ... */}
      <Suspense key={query + currentPage} fallback={<InvoicesTableSkeleton />}>
        <Table query={query} currentPage={currentPage} />
      </Suspense>
      {/* ... */}
    </div>
  );
}
```

:::

この場合、Server Functionsと`useActionState()`のみでは実現が難しいリロード復元やURLシェアが実現できます。上記例のように検索が主であるページにおいては、状態をURLに保存すること^[URLに状態を保存するには、[`location-state`↗︎](https://github.com/recruit-tech/location-state)や[`nuqs`↗︎](https://nuqs.dev/)などのライブラリの利用が便利です。]を検討すべきでしょう。`useActionState()`を使いつつ、状態をURLに保存することもできます。

一方サイドナビゲーションやcmd+kで開く検索モーダルのように、リロード復元やURLシェアをする必要がないケースでは、Server Functionsと`useActionState()`の実装が非常に役立つことでしょう。

### データ操作に伴う再レンダリング

ここで紹介したのはユーザー操作に伴うデータフェッチ、つまり**データ操作を伴わない**場合の設計パターンです。しかし、ユーザー操作にともなってデータを操作し、その後の結果を再取得したいこともあります。これはServer Actions^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]と`revalidatePath()`/`revalidateTag()`を組み合わせ実行することで実現できます。

これについては、後述の[データ操作とServer Actions](part_3_data_mutation)にて詳細を解説します。

---
title: "第2部 コンポーネント設計"
---

[RSCのRFC↗︎](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)には以下のような記述があります。

> The fundamental challenge was that React apps were client-centric and weren’t taking sufficient advantage of the server.
> _根本的な課題は、Reactアプリがクライアント中心で、サーバーを十分に活用していないことだった。_

Reactコアチームは、Reactが抱えていたいくつかの課題を個別の課題としてではなく、根本的には**サーバーを活用できていない**ことが課題であると考えました。そして、サーバー活用と従来のクライアント主体なReactの統合を目指し、設計されたアーキテクチャがReact Server Componentsです。

[第1部 データフェッチ](part_1)で解説してきた通り、特にデータフェッチに関しては従来よりシンプルでセキュアに実装できるようになったことで、ほとんどトレードオフなくコンポーネントにカプセル化することが可能となりました。一方コンポーネント設計においては、従来のクライアント主体のReactコンポーネント相当であるClient Componentsと、Server Componentsをうまく統合していく必要があります。

第2部ではRSCにおけるコンポーネント設計パターンを解説します。

---
title: "クライアントとサーバーのバンドル境界"
---

## 要約

`"use client"`や`"use server"`は実行環境を示すものではありません。これらはバンドラに**バンドル境界**を宣言するためのものです。

Server Bundleでのみ利用可能なモジュールは、`server-only`を使ってモジュールを保護しましょう。

:::message
本章の解説内容は、[Dan Abramov氏の記事↗︎](https://overreacted.io/what-does-use-client-do/)を参考にしています。より詳細に知りたい方は元記事をご参照ください。
:::

## 背景

RSCは多段階計算^[参考: [一言で理解するReact Server Components↗︎](https://zenn.dev/uhyo/articles/react-server-components-multi-stage)]アーキテクチャであり、サーバー側処理とクライアント側処理の2段階計算で構成されます。これはつまり、バンドルもサーバーバンドル（**Server Bundle**）とクライアントバンドル（**Client Bundle**^[Next.jsの内部的には、Client Bundleはブラウザで実行されるCSR用とNode.jsで実行されるSSR/SSG用それぞれのバンドルファイルが作成されるようです]）の2つに分けられることを意味し、Server ComponentsやServer FunctionsはServer Bundleに、Client ComponentsはClient Bundleに含められます。

多くの人は、`"use client"`や`"use server"`などのディレクティブがバンドルに関する重要なルールであることは知っています。しかし、これらのディレクティブの役割については「実行環境を示すためのもの」と誤解されることがよくあるようです^[`"use server"`に関する誤解を発端とした議論例: [Dan Abramov氏のBlueSkyでのやりとり↗︎](https://bsky.app/profile/danabra.mov/post/3lnw334g5jc24)]。実際には、これらのディレクティブは**実行環境を示すものではありません**。

## 設計・プラクティス

`"use client"`や`"use server"`は、**バンドル境界**を宣言するためのものです。

- `"use client"`:
  - **サーバー -> クライアント**のバンドル境界（**Client Boundary**）
  - サーバーへClient Componentsを公開する
- `"use server"`:
  - **クライアント -> サーバー**のバンドル境界（**Server Boundary**）
  - クライアントへServer Functionsを公開する

これにより、RSCでは2つのバンドルを1つのプログラムとして表現することができます。`"use client"`や`"use server"`は、RSCにおいて最も重要な責務を負ったルールです。これらの役割を正しく理解することは、Next.jsにおいても非常に重要です。

:::message alert

以下はよくある誤解です。

##### Q. Server Componentsには`"use server"`を付ける必要がある？

いいえ、前述の通り`"use server"`はServer Boundaryを宣言するためのものであり、Server Componentsを定義するためのものではありません。

##### Q. ではServer Componentsを定義するにはどうすればいい？

Next.jsでは、デフォルトでServer Componentsとなるので何も指定する必要はありません。

##### Q. Client ComponentsにServer Componentsを含めるにはどうすればいい？

Client Bundle内でServer Componentsを`import`することはできません。ただし、Client Componentsの`children`などにServer Componentsを渡すことは可能です。これは[Compositionパターン](part_2_composition_pattern)と呼ばれます。

:::

:::message
RSCにおけるバンドラの役割については、[uhyoさんの資料↗︎](https://speakerdeck.com/uhyo/rscnoshi-dai-nireacttohuremuwakunojing-jie-wotan-ru?slide=16)で詳しく解説されています。興味のある方はこちらをご参照ください。
:::

### モジュールの依存関係とバンドル境界

例として、ユーザー情報の編集ページで考えてみましょう。このページの`page.tsx`は、以下のような依存関係で構成されていると仮定します。

```
page.tsx
├── user-fetcher.ts
└── user-profile-form.tsx
    └── submit-button.tsx
```

以下はこれらのファイルの実装イメージです。

:::details 各ファイルの実装イメージ

```tsx:page.tsx
import { getUser } from "./user-fetcher";
import { UserProfileForm } from "./user-profile-form";

export default async function Page() {
  const user = await getUser();

  return (
    <div>
      <h1>User Profile</h1>
      <UserProfileForm defaultUser={user} />
    </div>
  );
}
```

```tsx:user-profile-form.tsx
"use client";

import { SubmitButton } from "./submit-button";
// ...省略...

export function UserProfileForm({ defaultUser }: { defaultUser: UserProfile }) {
  // ...省略...
  const onClick = () => {
    // ...省略...
  };

  return (
    <form>
      {/* ...省略... */}
      <SubmitButton onClick={onClick} />
    </form>
  );
}
```

```tsx:submit-button.tsx
export function SubmitButton({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>submit</button>;
}
```

```ts:user-fetcher.ts
export async function getUser() {
  // ...省略...
}
```

:::

これらのモジュールの依存関係をツリーで図示すると、以下のようになります。

![RSCのバンドル境界](/images/nextjs-basic-principle/rsc-bundle-boundary-1.png)

`submit-button.tsx`は`"use client"`を含みませんが、`"use client"`を含む`user-profile-form.tsx`から`import`されているため、Client Bundleに含まれます。このように、モジュールの依存関係にはバンドルの境界が存在し、`"use client"`はClient Boundaryを担います。

ここにさらに、フォームのサブミット時に呼び出されるServer Functionsを含む`update-profile-action.ts`を追加すると、以下のようになります。

:::details `update-profile-action.ts`の実装イメージ

```diff:user-profile-form.tsx
 // ...省略...
+ import { updateProfile } from "./update-profile-action";

 export function UserProfileForm({ defaultUser }: { defaultUser: UserProfile }) {
   // ...省略...
-  const onClick = () => {
+  const onClick = async () => {
+      await updateProfile(formData);
     // ...省略...
   };

   // ...省略...
 }
```

```ts:update-profile-action.ts
export async function updateProfile(formData: FormData) {
  "use server";

  // ...省略...
}
```

:::

![RSCのバンドル境界](/images/nextjs-basic-principle/rsc-bundle-boundary-2.png)

このように、`"use server"`はServer Boundaryを担います。

:::message
**Client Boundaryとなる**Client ComponentsはBundleを跨ぐため、受け取るPropsは原則として[Reactがserialize可能なもの↗︎](https://ja.react.dev/reference/rsc/use-client#serializable-types)である必要があります。一方**Client Boundaryでない**Client Componentsが受け取れるPropsは特に制約がありません。
:::

### 「2つの世界、2つのドア」

Dan Abramov氏は[前述の記事↗︎](https://overreacted.io/what-does-use-client-do/#two-worlds-two-doors)にて、`"use client"`と`"use server"`の役割を「2つの世界、2つのドア」という言葉で説明しています。RSCにはServer BundleとClient Bundleという2つの世界があり、これらの世界を開くドアが`"use client"`と`"use server"`です。

![境界とディレクティブ](/images/nextjs-basic-principle/rsc-layer.png)

## トレードオフ

### ファイル単位の`"use server"`による予期せぬエンドポイントの公開

`"use server"`は関数単位でもファイル単位でも宣言が可能です。ファイル単位で`"use server"`を宣言した場合、`export`された全ての関数はServer Functionsとして扱われます。これにより、意図せず関数がエンドポイントとして公開される可能性があるので、注意しましょう。

詳しくは以下の記事で解説されているので、ご参照ください。

https://zenn.dev/moozaru/articles/b0ef001e20baaf

### `server-only`

Server Bundleでのみ利用可能なモジュールを実装することは、よくあるユースケースです。このような場合には[`server-only`↗︎](https://www.npmjs.com/package/server-only)を使うことで、モジュールがServer Bundleでのみ利用されるよう保護できます。

```tsx
import "server-only";
```

もしClient Bundle内で`import "server-only"`を含むモジュールが見つかった場合、Next.jsはビルドできずエラーとなります。

逆に、Client Bundleでのみ利用可能なモジュールを実装する場合には、[`client-only`↗︎](https://www.npmjs.com/package/client-only)を利用することでClient Bundleでのみ利用されるよう保護できます。

---
title: "Client Componentsのユースケース"
---

## 要約

Client Componentsを使うべき代表的なユースケースを覚えておきましょう。

- [#クライアントサイド処理](#クライアントサイド処理)
- [#サードパーティコンポーネント](#サードパーティコンポーネント)
- [#RSC Payload転送量の削減](#rsc-payload転送量の削減)

## 背景

[第1部 データフェッチ](part_1)では、データフェッチ観点を中心にServer Componentsの設計パターンについて解説してきました。Next.jsにおけるコンポーネント全体の設計は、Server Componentsを中心とした設計にClient Componentsを適切に組み合わせていく形で行います。

そのためには、そもそもいつClient Componentsにオプトインすべきなのか適切に判断できることが重要です。

## 設計・プラクティス

筆者がClient Componentsを利用すべきだと考える代表的な場合は大きく以下の3つです。

### クライアントサイド処理

最もわかりやすくClient Componentsが必要な場合は、クライアントサイド処理を必要とする場合です。以下のような場合が考えられます。

- `onClick()`や`onChange()`といったイベントハンドラの利用
- 状態hooks(`useState()`や`useReducer()`など)やライフサイクルhooks(`useEffect()`など)の利用
- ブラウザAPIの利用

```tsx
"use client";

import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

### サードパーティコンポーネント

Client Componentsを提供するサードパーティライブラリがRSCに未対応な場合は、利用者側でClient Boundaryを明示しなければならないことがあります。この場合は`"use client"`を指定してre-exportするか、利用者側で`"use client"`を指定する必要があります。

```tsx :app/_components/accordion.tsx
"use client";

import { Accordion } from "third-party-library";

export default Accordion;
```

```tsx :app/_components/side-bar.tsx
"use client";

import { Accordion } from "third-party-library";

export function SideBar() {
  return (
    <div>
      <Accordion>{/* ... */}</Accordion>
    </div>
  );
}
```

### RSC Payload転送量の削減

3つ目は[RSC Payload↗︎](https://nextjs.org/docs/app/getting-started/server-and-client-components#on-the-server)の転送量を減らしたい場合です。Client Componentsは当然ながらクライアントサイドでも実行されるので、Client Componentsが多いほどJavaScriptバンドルサイズは増加します。一方Server ComponentsはRSC Payloadとして転送されるため、Server ComponentsがレンダリングするReactElementや属性が多いほど転送量が多くなります。

つまり、Client ComponentsのJavaScript転送量とServer ComponentsのRSC Payload転送量は**トレードオフ**になります。

Client Componentsを含むJavaScriptバンドルは1回しかロードされませんが、Server ComponentsはレンダリングされるたびにRSC Payloadが転送されます。そのため、繰り返しレンダリングされるコンポーネントはRSC Payloadの転送量を削減する目的でClient Componentsにすることが望ましい場合があります。

例えば以下の`<Product>`について考えてみます。

```tsx
export async function Product() {
  const product = await fetchProduct();

  return (
    <div class="... /* 大量のtailwindクラス */">
      <div class="... /* 大量のtailwindクラス */">
        <div class="... /* 大量のtailwindクラス */">
          <div class="... /* 大量のtailwindクラス */">
            {/* `product`参照 */}
          </div>
        </div>
      </div>
    </div>
  );
}
```

hooksなども特になく、ただ`product`を取得・参照しているのみです。しかしこのデータを参照してるReactElementの出力結果サイズが大きいと、RSC Payloadの転送コストが大きくなりパフォーマンス劣化を引き起こす可能性があります。特に低速なネットワーク環境においてページ遷移を繰り返す際などには影響が顕著になりがちです。

このような場合においてはServer Componentsではデータフェッチのみを行い、ReactElement部分はClient Componentsに分離することでRSC Payloadの転送量を削減することができます。

```tsx
export async function Product() {
  const product = await fetchProduct();

  return <ProductPresentaional product={product} />;
}
```

```tsx
"use client";

export function ProductPresentaional({ product }: { product: Product }) {
  return (
    <div class="... /* 大量のtailwindクラス */">
      <div class="... /* 大量のtailwindクラス */">
        <div class="... /* 大量のtailwindクラス */">
          <div class="... /* 大量のtailwindクラス */">
            {/* `product`参照 */}
          </div>
        </div>
      </div>
    </div>
  );
}
```

## トレードオフ

### Client Boundaryと暗黙的なClient Components

[クライアントとサーバーのバンドル境界](part_2_bundle_boundary)で解説したように`"use client"`はバンドル境界を定義するものであり、Client Bundleに含まれるコンポーネントは**全てClient Components**として扱われます。

上位のコンポーネントでClient Boundaryを宣言してしまうと下層でServer Componentsを含むことができなくなってしまい、RSCのメリットをうまく享受できなくなってしまうケースが散見されます。このようなケースへの対応は次章の[Compositionパターン](part_2_composition_pattern)で解説します。

### Server ComponentsからClient Componentsへ渡せるProps

Server ComponentsはClient Componentsを含むことができますが、これはServer BundleからClient Bundleへとバンドル境界を跨ぐため、渡せるPropsは、[Reactがserialize可能なもの↗︎](https://ja.react.dev/reference/rsc/use-client#serializable-types)に限られます。

```tsx
export async function MyServerComponent() {
  // ...

  return <ClientComponent data={/* Reactがserialize可能なもののみ渡せる */} />;
}
```

---
title: "Compositionパターン"
---

## 要約

Compositionパターンを駆使して、Server Componentsを中心に組み立てたコンポーネントツリーからClient Componentsを適切に切り分けましょう。

## 背景

[第1部 データフェッチ](part_1)で述べたように、RSCのメリットを活かすにはServer Components中心の設計が重要となります。そのため、Client Componentsは**適切に分離・独立**していることが好ましいですが、これを実現するにはClient Componentsの依存関係における以下2つの制約を考慮しつつ設計する必要があります。

:::message
以下は[クライアントとサーバーのバンドル境界](part_2_bundle_boundary)で解説した内容と重複します。
:::

### Client Bundleはサーバーモジュールを`import`できない

Client Bundle^[RSCにおいて、Client Componentsが含まれるバンドルを指します。]はServer Componentsはじめサーバーモジュールを`import`できません。

そのため、以下のような実装はできません。

```tsx
"use client";

import { useState } from "react";
import { UserInfo } from "./user-info"; // Server Components

export function SideMenu() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <UserInfo />
      <div>
        <button type="button" onClick={() => setOpen((prev) => !prev)}>
          toggle
        </button>
        <div>...</div>
      </div>
    </>
  );
}
```

この制約に対し唯一例外となるのが`"use server"`が付与されたファイルや関数、つまり [Server Functions↗︎](https://ja.react.dev/reference/rsc/server-functions)です。

::::details Server Functionsの実装例

```ts :create-todo.ts
"use server";

export async function createTodo() {
  // サーバーサイド処理
}
```

```tsx :create-button.tsx
"use client";

import { createTodo } from "./create-todo"; // 💡Server Functionsならimportできる

export function CreateButton({ children }: { children: React.ReactNode }) {
  return <button onClick={createTodo}>{children}</button>;
}
```

:::message
Server FunctionsはClient Bundleから普通の関数のように実行することが可能ですが、実際には当然通信処理が伴うため、引数や戻り値には[Reactがserialize可能なもの↗︎](https://ja.react.dev/reference/rsc/use-server#serializable-parameters-and-return-values)のみを利用できます。
:::

::::

### Client Boundary

`"use client"`が記述されたClient Boundary^[サーバー -> クライアントのバンドル境界を指します。]となるモジュールから`import`されるモジュールとその子孫は、**暗黙的に全てClient Bundle**に含まれます。そのため、定義されたコンポーネントは全てClient Componentsとして実行可能でなければなりません。

:::message alert
以下はよくある誤解です。

##### Q. `"use client"`を宣言したモジュールのコンポーネントだけがClient Components？

`"use client"`はClient Boundaryを宣言するためのものであり、Client Bundleに含まれるコンポーネントは全てClient Componentsとして扱われます。

##### Q. 全てのClient Componentsに`"use client"`が必要？

Client Boundaryとして扱うことがないなら`"use client"`は不要です。Client Bundleに含まれることを保証したいなら、[`client-only`↗︎](https://www.npmjs.com/package/client-only)を利用しましょう。

:::

## 設計・プラクティス

前述の通り、RSCでServer Componentsの設計を活かすにはClient Componentsを独立した形に切り分けることが重要となります。

これには大きく以下2つの方法があります。

### コンポーネントツリーの末端をClient Componentsにする

1つは、コンポーネントツリーの**末端をClient Componentsにする**というシンプルな方法です。Client Boundaryを下層に限定するとも言い換えられます。

例えば検索バーを持つヘッダーの場合、ヘッダー全体ではなく検索バー部分をClient Boundaryとし、ヘッダー自体はServer Componentsに保つといった方法です。

```tsx :header.tsx
import { SearchBar } from "./search-bar"; // Client Components

// page.tsxなどのServer Componentsから利用される
export function Header() {
  return (
    <header>
      <h1>My App</h1>
      <SearchBar />
    </header>
  );
}
```

### Compositionパターンを活用する

上記の方法はシンプルな解決策ですが、どうしても上位のコンポーネントをClient Componentsにする必要がある場合もあります。その際には**Compositionパターン**を活用して、Client Componentsを分離することが有効です。

前述の通りClient BundleはServer Componentsを`import`することができませんが、これはモジュールツリーにおける制約であり、コンポーネントツリーとしてはClient Componentsの`children`などのpropsにServer Componentsを渡すことで、レンダリングが可能です^[参考: [公式ドキュメント↗︎](https://ja.react.dev/reference/rsc/use-client#why-is-copyright-a-server-component)]。

前述の`<SideMenu>`の例を書き換えてみます。

```tsx :side-menu.tsx
"use client";

import { useState } from "react";

// `children`に`<UserInfo>`などのServer Componentsを渡すことが可能！
export function SideMenu({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);

  return (
    <>
      {children}
      <div>
        <button type="button" onClick={() => setOpen((prev) => !prev)}>
          toggle
        </button>
        <div>...</div>
      </div>
    </>
  );
}
```

```tsx :page.tsx
import { UserInfo } from "./user-info"; // Server Components
import { SideMenu } from "./side-menu"; // Client Components

/**
 * Client Components(`<SideMenu>`)の子要素として
 * Server Components(`<UserInfo>`)を渡せる
 */
export function Page() {
  return (
    <div>
      <SideMenu>
        <UserInfo />
      </SideMenu>
      <main>{/* ... */}</main>
    </div>
  );
}
```

`<SideMenu>`の`children`がServer Componentsである`<UserInfo />`となっています。これがいわゆるCompositionパターンと呼ばれる実装パターンです。

## トレードオフ

### 「後からComposition」の手戻り

Compositionパターンを駆使すればServer Componentsを中心にしつつ、部分的にClient Componentsを組み込むことが可能です。しかし、上位のコンポーネントにClient Boundaryを宣言し、後からCompositionパターンを導入しようとすると、Client Componentsの設計を大幅に変更せざるを得なくなったりServer Components中心な設計から逸脱してしまう可能性があります。

このような手戻りを防ぐためのテクニックとして、次章では[UIをツリーに分解する](part_2_container_1st_design)設計手順について解説します。

---
title: "UIをツリーに分解する"
---

## 要約

Reactの基本的な設計思想は、UI^[ここでのUIとは、データフェッチ等を含むReactコンポーネントで行う全てのことを含みます。]をコンポーネントのツリーで設計することです。ページやレイアウトなどの実装は、**UIをツリーに分解する**ことから始めましょう。これにより、データフェッチコロケーションやCompositionパターンの早期適用を目指します。

:::message
本章の内容は、React公式ドキュメントの[UIをツリーに分解する↗︎](https://ja.react.dev/learn/understanding-your-ui-as-a-tree#your-ui-as-a-tree)の理解が前提です。自信のない方はこちらを先にご参照ください。
:::

## 背景

[第1部 データフェッチ](part_1)でServer Componentsの設計パターンを、[第2部 コンポーネント設計](part_2)ではここまでClient Componentsの設計パターンを解説してきました。特に、[データフェッチ コロケーション](part_1_colocation)や[Compositionパターン](part_2_composition_pattern)は、後から適用しようとすると大きな手戻りを生む可能性があるため、早期から考慮して設計することが重要です。

## 設計・プラクティス

データフェッチコロケーションとCompositionパターンを早期適用するには、トップダウンな設計が効果的です。レイアウトやページといったUIを**ツリーに分解する**ことから始め、コンポーネントツリーの実装、各コンポーネントの詳細実装という流れで実装しましょう。

### 「大きなコンポーネント」と「小さなコンポーネント」

ReactではUIをコンポーネントとして表現します。ページやレイアウトなどのUIは「大きなコンポーネント」であり、「小さなコンポーネント」を組み合わせて^[ここでの「大きい」「小さい」とは、コンポーネントの位置関係を表すものです。コンポーネントの実装のサイズ（行数）ではありません。]実装します。RSCにおいてはServer Componentsという新たな種類のコンポーネントを組み合わせることができるようになりましたが、「小さなコンポーネント」を組み合わせて「大きなコンポーネント」を表現するという基本的な設計思想は変わりません。

「大きなコンポーネント」はボトムアップに実装すると手戻りが多くなりやすいため、筆者はトップダウンに設計することを推奨します。

### 実装手順

具体的には、以下のように進めることを推奨します。

1. 設計: **UIをツリーに分解する**
2. 仮実装: コンポーネントのツリーを仮実装
3. 実装: 各コンポーネントの詳細実装
   a. Server Componentsを実装
   b. Shared/Client Componentsを実装

:::message
最初に決めたツリー構造に固執する必要はありません。実装を進める中でツリーを見直すことも重要です。
:::

### 実装例

以下のようなブログ記事画面を例として考えてみます。

![UIをツリー構造に分解する](/images/nextjs-basic-principle/blog-ui-example.png =400x)

この画面は以下のような要素で構成されています。

- ブログ記事情報
- 著者情報
- コメント一覧

これらのデータの取得には、以下のAPIを利用するものとします。

- PostAPI: 投稿IDをもとにブログ記事情報を取得するAPI
- UserAPI: ユーザーIDをもとにユーザー情報を取得するAPI
- CommentsAPI: 投稿IDをもとにコメント一覧を取得するAPI

#### 1. UIをツリー構造に分解する

UIを画面の要素が依存するデータを元にツリーに分解します。

![APIの依存関係](/images/nextjs-basic-principle/component-tree-example.png)

#### 2. コンポーネントのツリーを仮実装

上記の図をもとに、分解したUIの各要素をServer Componentsとして仮実装します。ここでデータフェッチするコンポーネントを`{Name}Container`という命名で仮実装します^[`Container`という命名は、[Container/Presentationalパターン](part_2_container_presentational_pattern)を元にしています。]。

```tsx:/posts/[postId]/page.tsx
export default async function Page(props: {
  params: Promise<{ postId: string }>;
}) {
  const { postId } = await props.params;

  return (
    <div className="flex flex-col gap-4">
      <PostContainer postId={postId}>
        <UserProfileContainer postId={postId} />
      </PostContainer>
      <CommentsContainer postId={postId} />
    </div>
  );
}
```

#### 3. 各コンポーネントの詳細実装

以降は、仮実装となっているContainer Componentsの詳細な実装を行い、UIを完成させます。

各コンポーネントの詳細な実装は主題ではないため、本章では省略します。

::::details 各コンポーネントの実装イメージ

以下はContainer Componentsの実装イメージです。データフェッチ層などの実装は省略しています。

```tsx
// `/posts/[postId]/_containers/post/container.tsx`
export async function PostContainer({
  postId,
  children,
}: {
  postId: string;
  children: React.ReactNode;
}) {
  const post = await getPost(postId); // Request Memoization

  return (
    <div>
      {/* ...省略... */}
      {children}
      {/* ...省略... */}
    </div>
  );
}

// `/posts/[postId]/_containers/user-profile/container.tsx`
export async function UserProfileContainer({ postId }: { postId: string }) {
  const post = await getPost(postId); // Request Memoization
  const user = await getUser(post.authorId);

  return <div>{/* ...省略... */}</div>;
}

// `/posts/[postId]/_containers/comments/container.tsx`
export async function CommentsContainer({ postId }: { postId: string }) {
  const comments = await getComments(postId);

  return (
    <div>
      {comments.map((comment) => (
        <div key={comment.id}>{/* ...省略... */}</div>
      ))}
    </div>
  );
}

async function CommentItemContainer({ comment }: { comment: Comment }) {
  const user = await getUser(comment.authorId); // `getUser`は内部的にDataLoaderを利用

  return <div>{/* ...省略... */}</div>;
}
```

::::

## トレードオフ

### 重複するデータフェッチやN+1データフェッチ

UIを「小さなコンポーネント」に分解し、末端のコンポーネントでデータフェッチを行うことは重複リクエストのリスクが伴います。[Request Memoization](part_1_request_memoization)の章で解説したように、Next.jsではRequest Memoizationによってレンダリング中の同一リクエストを排除するため、データフェッチ層の設計が重要です。

また、配列を扱う際には[DataLoader](part_1_data_loader)を利用することで、N+1データフェッチを解消することができます。

---
title: "Container/Presentationalパターン"
---

## 要約

データ取得はContainer Components、データ参照はPresentational Componentsに分離し、テスト容易性を向上させましょう。

:::message
本章の解説内容は、[Quramyさんの記事↗︎](https://quramy.medium.com/react-server-component-%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8-container-presentation-separation-7da455d66576)を参考にしています。ほとんど要約した内容となりますので、より詳細に知りたい方は元記事をご参照ください。
:::

## 背景

Reactコンポーネントのテストといえば[React Testing Library↗︎](https://testing-library.com/docs/react-testing-library/intro/)(RTL)や[Storybook↗︎](https://storybook.js.org/)などを利用することが主流ですが、本書執筆時点でこれらのRSC対応状況は芳しくありません。

### React Testing Library

RTLは現状[Server Componentsに未対応↗︎](https://github.com/testing-library/react-testing-library/issues/1209)で、将来的にサポートするようなコメントも見られますが時期については不明です。

具体的には非同期なコンポーネントを`render()`することができないため、以下のようにServer Componentsのデータフェッチに依存した検証はできません。

```tsx
test("random Todo APIより取得した`dummyTodo`がタイトルとして表示される", async () => {
  // mswの設定
  server.use(
    http.get("https://dummyjson.com/todos/random", () => {
      return HttpResponse.json(dummyTodo);
    }),
  );

  await render(<RandomTodo />); // `<RandomTodo>`はServer Components

  expect(
    screen.getByRole("heading", { name: dummyTodo.title }),
  ).toBeInTheDocument();
});
```

:::message
執筆時点では開発中ですが、将来的には[vitest-plugin-rsc↗︎](https://github.com/kasperpeulen/vitest-plugin-rsc)などを使うことでRSCのテストが可能になるかもしれません。
:::

### Storybook

一方、Storybookはexperimentalで[RSCに対応↗︎](https://storybook.js.org/blog/storybook-react-server-components/)していますが、内部的にはこれは非同期なClient Componentsをレンダリングしているにすぎず、大量のmockを必要とするため、実用性に疑問が残ります。

```tsx
export default { component: DbCard };

export const Success = {
  args: { id: 1 },
  parameters: {
    moduleMock: {
      // サーバーサイド処理の分`mock`が冗長になる
      mock: () => {
        const mock = createMock(db, "findById");
        mock.mockReturnValue(
          Promise.resolve({
            name: "Beyonce",
            img: "https://blackhistorywall.files.wordpress.com/2010/02/picture-device-independent-bitmap-119.jpg",
            tel: "+123 456 789",
            email: "b@beyonce.com",
          }),
        );
        return [mock];
      },
    },
  },
};
```

## 設計・プラクティス

前述の状況を踏まえると、テスト対象となるServer Componentsは「テストしにくいデータフェッチ部分」と「テストしやすいHTMLを表現する部分」で分離しておくことが望ましいと考えられます。

このように、データを提供する層とそれを表現する層に分離するパターンは**Container/Presentationalパターン**の再来とも言えます。

### 従来のContainer/Presentationalパターン

Container/Presentationalパターンは元々、Flux全盛だったReact初期に提唱された設計手法です。データの読み取り・振る舞い(主にFluxのaction呼び出しなど)の定義をContainer Componentsが、データを参照し表示するのはPresentational Componentsが担うという責務分割がなされていました。

:::message
興味のある方は、当時Container/Presentationalパターンを提唱した[Dan Abramov氏の記事↗︎](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)をご参照ください。
:::

### RSCにおけるContainer/Presentationalパターン

RSCにおけるContainer/Presentationalパターンは従来のものとは異なり、Container Componentsはデータフェッチなどのサーバーサイド処理のみを担います。一方Presentational Componentsは、データフェッチを含まない**Shared Components**もしくはClient Componentsを指します。

| Components     | React初期                                           | RSC時代                                                           |
| -------------- | --------------------------------------------------- | ----------------------------------------------------------------- |
| Container      | 状態参照、状態変更関数の定義                        | Server Components上でのデータフェッチなどの**サーバーサイド処理** |
| Presentational | `props`を参照してReactElementを定義する純粋関数など | **Shared Components/Client Components**                           |

Shared Componentsはクライアントまたはサーバーでのみ使える機能に依存せず、`"use client"`もないモジュールで定義されるコンポーネントを指します。このようなコンポーネントは、Client Bundle^[参考: [クライアントとサーバーのバンドル境界](part_2_bundle_boundary)]においてはClient Componentsとして扱われ、Server BundleにおいてはServer Componentsとして扱われます。

```tsx
// `"use client"`がないモジュール
export function CompanyLinks() {
  return (
    <ul>
      <li>
        <a href="/about">About</a>
      </li>
      <li>
        <a href="/contact">Contact</a>
      </li>
    </ul>
  );
}
```

:::message
上記のようにClient Boundaryでないコンポーネントで、Client Bundleに含まれることを保証したい場合には[`client-only`↗︎](https://www.npmjs.com/package/client-only)パッケージを利用しましょう。
:::

Client ComponentsやShared Componentsは従来通りRTLやStorybookで扱うことができるので、**テスト容易性が向上**します。一方Container Componentsはこれらのツールでレンダリング・テストすることは現状難しいですが、`await ArticleContainer({ id })`のように単なる関数として実行することでテストが可能です。

### 実装例

[UIをツリーに分解する](part_2_container_1st_design)の実装例同様、ブログ記事情報の取得と表示を担うコンポーネントをContainer/Presentationalパターンで実装してみます。

#### Container Componentsの実装とテスト

Container Componentsではブログ記事情報の取得を行い、Presentational Componentsにデータを渡します。

```tsx:/posts/[postId]/_containers/post/container.tsx
export async function PostContainer({
  postId,
  children,
}: {
  postId: string;
  children: React.ReactNode;
}) {
  const post = await getPost(postId); // Request Memoization

  return <PostPresentation post={post}>{children}</PostPresentation>;
}
```

:::message

- `getPost(postId)`のようにデータフェッチ層を分離することで、[Request Memoization](part_1_request_memoization)による重複リクエスト排除を活用しやすくなります。
- 上記例ではContainerとPresentationalが1対1となっていますが、必ずしも1対1になるとは限りません。

:::

前述の通り、`<PostContainer>`のような非同期コンポーネントはRTLで`render()`することができないため、コンポーネントとしてテストすることはできません。そのため、単なる関数として実行して戻り値を検証します。

以下は簡易的なテストケースの実装例です。

```ts:/posts/[postId]/_containers/post/container.test.tsx
describe("PostAPIよりデータ取得成功時", () => {
  test("PostPresentationにAPIより取得した値が渡される", async () => {
    // mswの設定
    server.use(
      http.get("https://dummyjson.com/posts/postId", () => {
        return HttpResponse.json(post);
      }),
    );

    const { type, props } = await PostContainer({ postId: "1" });

    expect(type).toBe(PostPresentation);
    expect(props.post).toEqual(post);
  });
});
```

このように、コンポーネントを通常の関数のように実行すると`type`や`props`を得ることができるので、これらを元に期待値通りかテストすることができます。

ただし、上記のようなテストは`ReactElement`の構造に強く依存してしまいFlaky(壊れやすい)なテストになってしまいます。そのため、実際には[こちらの記事↗︎](https://quramy.medium.com/react-server-component-%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8-container-presentation-separation-7da455d66576#:~:text=%E3%81%8A%E3%81%BE%E3%81%912%3A%20Container%20%E3%82%B3%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%8D%E3%83%B3%E3%83%88%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%81%A8%20JSX%20%E3%81%AE%E6%A7%8B%E9%80%A0)にあるように、`ReactElement`を扱うユーティリティの作成やスナップショットテストなどを検討すると良いでしょう。

```tsx
describe("PostAPIよりデータ取得成功時", () => {
  test("PostPresentationにAPIより取得した値が渡される", async () => {
    // mswの設定
    server.use(
      http.get("https://dummyjson.com/posts/postId", () => {
        return HttpResponse.json(post);
      }),
    );

    const container = await PostContainer({ postId: "1" });

    expect(
      getProps<typeof PostPresentation>(container, PostPresentation),
    ).toEqual({
      post,
    });
  });
});
```

#### Presentational Componentsの実装とテスト

一方Presentational Componentsは、データを受け取って表示するだけのシンプルなコンポーネントになります。

```tsx
export function PostPresentation({ post }: { post: Post }) {
  return (
    <>
      <h1>{post.title}</h1>
      <pre>
        <code>{JSON.stringify(post, null, 2)}</code>
      </pre>
    </>
  );
}
```

必要に応じて`"use client"`を宣言し、Client Componentsにすることもできます。

このようなコンポーネントは従来同様RTLやStorybookを使って、容易にテストできます。

```tsx
test("`post`として渡された値がタイトルとして表示される", () => {
  const post = {
    title: "test post",
  };
  render(<PostPresentation post={post} />);

  expect(
    screen.getByRole("heading", {
      name: "test post",
      level: 1,
    }),
  ).toBeInTheDocument();
});
```

### Container単位のディレクトリ構成例

Next.jsはファイルコロケーションを強く意識して設計されており、[Route Segment↗︎](https://nextjs.org/docs/app/getting-started/layouts-and-pages#creating-a-nested-route)で利用するコンポーネントや関数もできるだけコロケーションすることが推奨^[参考: [公式ドキュメント↗︎](https://nextjs.org/docs/app/getting-started/project-structure#colocation)]されます。上記手順で得られたページやレイアウトを構成するContainer Componentsも、同様にコロケーションすることが望ましいと考えられます。

以下は、筆者が推奨するディレクトリ構成の例です。[Private Folder↗︎](https://nextjs.org/docs/app/getting-started/project-structure#private-folders)を利用して、Container単位で`_containers`ディレクトリにコロケーションします。

```
/posts/[postId]
├── page.tsx
├── layout.tsx
└── _containers
    ├── post
    │  ├── index.tsx // Container Componentsをre export
    │  ├── container.tsx
    │  ├── presentational.tsx
    │  └── ... // その他のコンポーネントやUtilityなど
    └── user-profile
       ├── index.tsx // Container Componentsをre export
       ├── container.tsx
       ├── presentational.tsx
       └── ... // その他のコンポーネントやUtilityなど
```

コロケーションしたファイルは、外部から参照されることを想定した実質的にPublicなファイルと、Privateなファイルに分けることができます。上記の例では、`index.tsx`でContainer Componentsをre exportすることを想定しています。

## トレードオフ

### エコシステム側が将来対応する可能性

本章では、RSCに対するテストやStorybookの対応が未成熟であることを前提にしつつ、テスト容易性を向上するための手段としてContainer/Presentationalパターンが役に立つことを主張しました。しかし、今後エコシステムの状況が変わればより容易にテストできるようになることがあるかもしれません。その場合、Container/Presentationalパターンは変化するか不要になる可能性もあります。

### 広すぎるexport

Presentational ComponentsはContainer Componentsの実装詳細と捉えることもできるので、本来プライベート定義として扱うことが好ましいと考えられます。[Container単位のディレクトリ構成例](#container単位のディレクトリ構成例)では、Presentational Componentsは`presentational.tsx`で定義されます。

```
_containers
├── <Container Name> // e.g. `post-list`, `user-profile`
│  ├── index.tsx // Container Componentsをre export
│  ├── container.tsx
│  ├── presentational.tsx
│  └── ...
└── ...
```

上記の構成では`<Container Name>`の外から参照されるモジュールは`index.tsx`のみの想定です。ただ実際には、`presentational.tsx`で定義したコンポーネントもプロジェクトのどこからでも参照することができます。

このように、同一ディレクトリにおいてのみ利用することを想定したモジュール分割においては、[eslint-plugin-import-access↗︎](https://github.com/uhyo/eslint-plugin-import-access)やbiomeの[`noPrivateImports`↗︎](https://biomejs.dev/linter/rules/no-private-imports/)を利用すると予期せぬ外部からの`import`を制限することができます。

上記のようなディレクトリ設計に沿わない場合でも、Presentational ComponentsはContainer Componentsのみが利用しうる**実質的なプライベート定義**として扱うようにしましょう。

---
title: "第3部 キャッシュ"
---

Next.jsには4層のキャッシュが存在し、デフォルトで積極的に利用されます。以下は[公式の表↗︎](https://nextjs.org/docs/app/guides/caching#overview)を翻訳したものです。

| Mechanism               | What              | Where  | Purpose                                    | Duration                                |
| ----------------------- | ----------------- | ------ | ------------------------------------------ | --------------------------------------- |
| **Request Memoization** | APIレスポンスなど | Server | React Component treeにおけるデータの再利用 | リクエストごと                          |
| **Data Cache**          | APIレスポンスなど | Server | ユーザーやデプロイをまたぐデータの再利用   | 永続的 (revalidate可)                   |
| **Full Route Cache**    | HTMLやRSC payload | Server | レンダリングコストの最適化                 | 永続的 (revalidate可)                   |
| **Router Cache**        | RSC Payload       | Client | ナビゲーションごとのリクエスト削減         | ユーザーセッション・時間 (revalidate可) |

:::message
Next.js v15で[キャッシュのデフォルト設定の見直し↗︎](https://nextjs.org/blog/next-15-rc#caching-updates)が行われ、従来ほど積極的なキャッシュは行われなくなりました。
現在実験的機能である[`cacheComponents`↗︎](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents)フラグでは、さらにサーバーサイドキャッシュはデフォルトで無効化され、`"use cache"`でキャッシュをオプトインする設計に変更することが検討されています。
:::

すでに[Request Memoization](part_1_request_memoization)については解説しましたが、これはNext.jsサーバーに対するリクエスト単位という非常に短い期間でのみ利用されるキャッシュであり、これが問題になることはほとんどないと考えられます。一方他の3つについてはもっと長い期間および広いスコープで利用されるため、開発者が意図してコントロールしなければ予期せぬキャッシュによるバグに繋がりかねません。そのため、Next.jsを利用する開発者にとってこれらの理解は非常に重要です。

第3部ではNext.jsにおけるキャッシュの考え方や仕様、コントロールの方法などを解説します。

---
title: "Static RenderingとFull Route Cache"
---

## 要約

Static Renderingでは、HTMLやRSC PayloadのキャッシュであるFull Route Cacheを生成します。Full Route Cacheは短い期間でrevalidate可能なので、ユーザー固有の情報を含まないようなページは積極的にFull Route Cacheを活用しましょう。

## 背景

Next.jsは、従来Pages Routerで[SSR↗︎](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG↗︎](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)という3つのレンダリングモデル^[[こちらの記事↗︎](https://vercel.com/blog/partial-prerendering-with-next-js-creating-a-new-default-rendering-model)より引用。レンダリング戦略とも呼ばれます。]をサポートしてきました。App Routerでは上記相当のレンダリングをサポートしつつ、revalidateがより整理され、SSGとISRを大きく区別せずまとめて**Static Rendering**、従来のSSR相当を**Dynamic Rendering**と呼称する形で[サーバー側レンダリングが再定義↗︎](https://nextjs.org/docs/app/getting-started/linking-and-navigating#server-rendering)されました。

| レンダリング          | タイミング            | Pages Routerとの比較 |
| --------------------- | --------------------- | -------------------- |
| **Static Rendering**  | build時やrevalidate後 | SSG・ISR相当         |
| **Dynamic Rendering** | ユーザーリクエスト時  | SSR相当              |

:::message
Server Componentsは[Soft Navigation↗︎](https://nextjs.org/docs/app/getting-started/linking-and-navigating#client-side-transitions)時も実行されるのでSSR・SSG・ISRと単純に比較できるものではないですが、ここでは簡略化して比較しています。
:::

App Routerは**デフォルトでStatic Rendering**となっており、**Dynamic Renderingはオプトイン**になっています。Dynamic Renderingにオプトインする方法は以下の通りです。

### Dynamic APIs

`cookies()`/`headers()`などの[Dynamic APIs↗︎](https://nextjs.org/docs/app/guides/caching#dynamic-apis)と呼ばれるAPIを利用すると、Dynamic Renderingとなります。

```ts
// page.tsx
import { cookies } from "next/headers";

export default async function Page() {
  const cookieStore = await cookies();
  const sessionId = cookieStore.get("session-id");

  return "...";
}
```

:::message
[Dynamic Routes↗︎](https://nextjs.org/docs/app/getting-started/project-structure#dynamic-routes)における[`searchParams` props↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional)はDynamic APIsの1つとして数えられており、参照するとDynamic Renderingになります。一方[`params` props↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/page#params-optional)は、参照するとデフォルトでDynamic Renderingになりますが、[generateStaticParams()↗︎](https://nextjs.org/docs/app/api-reference/functions/generate-static-params)を利用するなどするとStatic Renderingになるため、必ずしもDynamic Renderingになるとは限りません。
:::

### `cache: "no-store"`もしくは`next.revalidate: 0`な`fetch()`

[`fetch()`のオプション↗︎](https://nextjs.org/docs/app/api-reference/functions/fetch#optionscache)で`cache: "no-store"`を指定した場合や、`next: { revalidate: 0 }`を指定した場合、Dynamic Renderingとなります。

:::message alert
v14以前において、[`cache`オプション↗︎](https://nextjs.org/docs/app/api-reference/functions/fetch#optionscache)のデフォルトは`"force-cache"`でした。v15ではデフォルトでキャッシュが無効になるよう変更されていますが、デフォルトではStatic Renderingとなっています。Dynamic Renderingに切り替えるには明示的に`"no-store"`を指定する必要があるので、注意しましょう。
:::

```ts
// page.tsx
export default async function Page() {
  const res = await fetch("https://dummyjson.com/todos/random", {
    // 🚨Dynamic Renderingにするために`"no-store"`を明示
    cache: "no-store",
  });
  const todoItem: TodoItem = await res.json();

  return "...";
}
```

### Route Segment Config

[Route Segment Config↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)を利用してDynamic Renderingに切り替えることもできます。具体的には、`page.tsx`や`layout.tsx`に以下どちらかを設定することでDynamic Renderingを強制できます。

```tsx
// layout.tsx | page.tsx
export const dynamic = "force-dynamic";
```

```tsx
// layout.tsx | page.tsx
export const revalidate = 0; // 1以上でStatic Rendering
```

:::message alert
`layout.tsx`に設定したRoute Segment ConfigはLayoutが利用される下層ページにも適用されるため、注意しましょう。
:::

### `connection()`

末端のコンポーネントから利用者側にDynamic Renderingを強制したいが、`headers()`や`no-store`な`fetch()`を使っていない場合には、[`connection()`↗︎](https://nextjs.org/docs/app/api-reference/functions/connection)でDynamic Renderingに切り替えることができます。具体的には、[Prisma↗︎](https://www.prisma.io/)を使ったDBアクセス時などに有用でしょう。

```ts
import { connection } from "next/server";

export async function LeafComponent() {
  await connection();

  // DBアクセスなど

  return "...";
}
```

## 設計・プラクティス

Static Renderingは耐障害性・パフォーマンスに優れています。ユーザーリクエスト毎にレンダリングが必要なら前述の方法でDynamic Renderingにオプトインする必要がありますが、それ以外のケースについて、Next.jsでは**可能な限りStatic Renderingにする**ことが推奨されています。

Static Renderingのレンダリング結果であるHTMLやRSC Payloadのキャッシュは、[Full Route Cache↗︎](https://nextjs.org/docs/app/guides/caching#full-route-cache)と呼ばれています。Next.jsではStatic Renderingを活用するために、Full Route Cacheのオンデマンドrevalidateや時間ベースでのrevalidateといったよくあるユースケースをフォローし、SSGのように変更があるたびにデプロイが必要といったことがないように設計されています。

### オンデマンドrevalidate

[`revalidatePath()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[`revalidateTag()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)をServer Actions^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]や[Route Handlers↗︎](https://nextjs.org/docs/app/getting-started/route-handlers-and-middleware#route-handlers)で呼び出すことで、関連するData CacheやFull Route Cacheをrevalidateすることができます。

```ts
"use server";

import { revalidatePath } from "next/cache";

export async function action() {
  // ...

  revalidatePath("/products");
}
```

### 時間ベースrevalidate

Route Segment Configの[revalidate↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config#revalidate)を指定することでFull Route Cacheや関連するData Cacheを時間ベースでrevalidateすることができます。

```tsx
// layout.tsx | page.tsx
export const revalidate = 10; // 10s
```

:::message
重複になりますが、`layout.tsx`に`revalidate`を設定するとLayoutが利用される下層ページにも適用されるため、注意しましょう。
:::

例えば1秒などの非常に短い時間でも設定すれば、瞬間的に非常に多くのリクエストが発生したとしてもレンダリングは1回で済むため、バックエンドAPIへの負荷軽減や安定したパフォーマンスに繋がります。更新頻度が非常に高いページでもユーザー間で共有できる(=ユーザー固有の情報などを含まない)のであれば、設定を検討しましょう。

## トレードオフ

### 予期せぬDynamic Renderingとパフォーマンス劣化

Route Segment Configや`connection()`によってDynamic Renderingを利用する場合、開発者は明らかにDynamic Renderingを意識して使うのでこれらが及ぼす影響を見誤ることは少ないと考えられます。一方、Dynamic APIsは「cookieを利用したい」、`cache: "no-store"`な`fetch`は「Data Cacheを使いたくない」などの主目的が別にあり、これに伴って副次的にDynamic Renderingに切り替わるため、開発者は影響範囲に注意する必要があります。

特に、Data Cacheなどを適切に設定できていないとDynamic Renderingに切り替わった際にページ全体のパフォーマンス劣化につながる可能性があります。こちらについての詳細は後述の[Dynamic RenderingとData Cache](part_3_dynamic_rendering_data_cache)をご参照ください。

---
title: "Dynamic RenderingとData Cache"
---

## 要約

Dynamic Renderingなページでは、データフェッチ単位のキャッシュであるData Cacheを活用してパフォーマンスを最適化しましょう。

## 背景

[Static RenderingとFull Route Cache](part_3_static_rendering_full_route_cache)で述べた通り、Next.jsでは可能な限りStatic Renderingにすることが推奨されています。しかし、アプリケーションによってはユーザー情報を含むページなど、Dynamic Renderingが必要な場合も当然考えられます。

Dynamic Renderingはリクエストごとにレンダリングされるので、できるだけ早く完了する必要があります。この際最もボトルネックになりやすいのが**データフェッチ処理**です。

:::message
RouteをDynamic Renderingに切り替える方法は前の章の[Static RenderingとFull Route Cache](part_3_static_rendering_full_route_cache#背景)で解説していますので、そちらをご参照ください。
:::

## 設計・プラクティス

[Data Cache↗︎](https://nextjs.org/docs/app/guides/caching#data-cache)はデータフェッチ処理の結果をキャッシュするもので、サーバー側に永続化され**リクエストやユーザーを超えて共有**されます。

Dynamic RenderingではNext.jsサーバーへのリクエストごとにレンダリングを行いますが、その際必ずしも全てのデータフェッチを実行しなければならないとは限りません。ユーザー情報に紐づくようなデータフェッチとそうでないものを切り分けて、後者に対しData Cacheを活用することで、Dynamic Renderingの高速化やAPIサーバーの負荷軽減などが見込めます。

Data Cacheができるだけキャッシュヒットするよう、データフェッチごとに適切な設定を心がけましょう。

### Next.jsと`fetch()`

サーバー上で実行される`fetch()`は[Next.jsによって拡張↗︎](https://nextjs.org/docs/app/api-reference/functions/fetch#fetchurl-options)されており、Data Cacheに関するオプションが組み込まれています。デフォルトではキャッシュは無効ですが、第2引数のオプション指定によってキャッシュ挙動を変更することが可能です。

```ts
fetch(`https://...`, {
  cache: "force-cache", // or "no-store",
});
```

:::message alert
v14以前において、[`cache`オプション↗︎](https://nextjs.org/docs/app/api-reference/functions/fetch#optionscache)のデフォルトは`"force-cache"`でした。v15ではデフォルトでキャッシュが無効になるよう変更されていますが、デフォルトではStatic Renderingとなっています。Dynamic Renderingに切り替えるには明示的に`"no-store"`を指定する必要があるので、注意しましょう。
:::

```ts
fetch(`https://...`, {
  next: {
    revalidate: false, // or number,
  },
});
```

`next.revalidate`は文字通りrevalidateされるまでの時間を設定できます。

```ts
fetch(`https://...`, {
  next: {
    tags: [tagName], // string[]
  },
});
```

`next.tags`には配列で**tag**を複数指定することができます。これは後述の`revalidateTag()`によって指定したtagに関連するData Cacheをrevalidateする際に利用されます。

### `unstable_cache()`

`unstable_cache()`を使うことで、DBアクセスなどでもData Cacheを利用することが可能です。

:::message alert
`unstable_cache()`は将来的に[`"use cache"`↗︎](https://nextjs.org/docs/app/api-reference/directives/use-cache)に置き換えられる予定です。
:::

```tsx
import { getUser } from "./fetcher";
import { unstable_cache } from "next/cache";

const getCachedUser = unstable_cache(
  getUser, // DBアクセス
  ["my-app-user"], // key array
  {
    tags: ["users"], // cache revalidate tags
    revalidate: 60, // revalidate time(s)
  },
);

export default async function Component({ userID }) {
  const user = await getCachedUser(userID);
  // ...
}
```

### オンデマンドrevalidate

[Static RenderingとFull Route Cache](part_3_static_rendering_full_route_cache)でも述べた通り、[`revalidatePath()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[`revalidateTag()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)をServer Actions^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]や[Route Handlers↗︎](https://nextjs.org/docs/app/getting-started/route-handlers-and-middleware)で呼び出すことで、関連するData CacheやFull Route Cacheをrevalidateすることができます。

```ts
"use server";

import { revalidatePath } from "next/cache";

export async function action() {
  // ...

  revalidatePath("/products");
}
```

これらは特に何かしらのデータ操作が発生した際に利用されることを想定したrevalidateです。サイト内ではServer Actionsを、外部で発生したデータ操作に対してはRoute Handlersからrevalidateすることが推奨されます。

Next.jsでのデータ操作に関する詳細は、後述の[データ操作とServer Actions](part_3_data_mutation)にて解説します。

::::details 余談: Data Cacheと`revalidatePath()`の仕組み

Data Cacheにはデフォルトのtagとして、Route情報を元にしたタグが内部的に設定されており、`revalidatePath()`はこの特殊なタグを元に関連するData Cacheのrevalidateを実現しています。

より詳細にrevalidateの仕組みを知りたい方は、過去に筆者が調査した際の以下の記事をぜひご参照ください。

https://zenn.dev/akfm/articles/nextjs-revalidate

::::

## トレードオフ

### Data CacheのオプトアウトとDynamic Rendering

`fetch()`のオプションで`cahce: "no-store"`か`next.revalidate: 0`を設定することでData Cacheをオプトアウトすることができますが、これは同時にRouteが**Dynamic Renderingに切り替わる**ことにもなります。

これらを設定する時は本当にDynamic Renderingにしなければいけないのか、よく考えて設定しましょう。

---
title: "Router Cache"
---

## 要約

Router Cacheはクライアントサイドのキャッシュで、prefetch時やSoft Navigation時に更新されます。アプリケーション特性に応じてRouter Cacheの有効期間である`staleTimes`を適切に設定しつつ、適宜必要なタイミングでrevalidateしましょう。

## 背景

[Router Cache↗︎](https://nextjs.org/docs/app/guides/caching#client-side-router-cache)は、App Routerにおけるクライアントサイドキャッシュで、Server Componentsのレンダリング結果である[RSC Payload↗︎](https://nextjs.org/docs/app/getting-started/server-and-client-components#on-the-server)を保持しています。Router Cacheはprefetchやsoft navigation時に更新され、有効期間内であれば再利用されます。

v14.1以前のApp RouterではRouter Cacheの有効期間を開発者から設定することはできず、`<Link>`の`prefetch`指定などに基づいてキャッシュの有効期限は**30秒か5分**とされていました。これに対しNext.jsリポジトリの[Discussion↗︎](https://github.com/vercel/next.js/discussions/54075)上では、Router Cacheをアプリケーション毎に適切に設定できるようにして欲しいという要望が相次いでいました。

## 設計・プラクティス

Next.jsの[v14.2↗︎](https://nextjs.org/blog/next-14-2#caching-improvements)にて、Router Cacheの有効期間を設定する`staleTimes`がexperimentalで導入されました。これにより、開発者はアプリケーション特性に応じてRouter Cacheの有効期間を適切に設定することができるようになりました。

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    staleTimes: {
      dynamic: 10,
      static: 180,
    },
  },
};

export default nextConfig;
```

### `staleTimes`の設定

`staleTimes`の設定は[ドキュメント↗︎](https://nextjs.org/docs/app/api-reference/config/next-config-js/staleTimes)によると以下のように対応していることになります。

| 項目    | `<Link prefetch=?>` | デフォルト(v14) | デフォルト(v15) |
| ------- | ------------------- | --------------- | --------------- |
| dynamic | `undefined`         | 30s             | 0s              |
| static  | `true`              | 5m              | 5m              |

:::message
[#トレードオフ](#トレードオフ)にて後述しますが、実際にはRouter Cacheの動作は非常に複雑なためこの対応の限りではありません。
:::

多くの場合、変更を考えるべくは`dynamic`の方になります。v14では特に、デフォルトだと30sとなっているために多くの混乱が見られました。利用するNext.jsのバージョンごとの挙動に注意して、キャッシュの有効期限としてどこまで許容できるか考えて適切に設定しましょう。

### 任意のタイミングでrevalidate

`staleTimes`以外でRouter Cacheを任意に破棄するには、以下3つの方法があります。

- [`router.refresh()`↗︎](https://nextjs.org/docs/app/guides/caching#routerrefresh)
- Server Actions^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]で`revalidatePath()`/`revalidateTag()`
- Server Actionsで`cookies.set()`/`cookies.delete()`

Router Cacheを任意のタイミングで破棄したい多くのユースケースは、ユーザーによるデータ操作時です。Server Actions内で`revalidatePath()`や`cookies.set()`を呼び出しているなら特に追加実装する必要はありません。一方これらを呼び出していない場合には、データ操作のsubmit完了後にClient Components側で`router.refresh()`を呼び出すなどの対応を行いましょう。

特に`revalidatePath()`/`revalidateTag()`はサーバー側キャッシュだけでなくRouter Cacheにも影響を及ぼすことは直感的ではないので、よく覚えておきましょう。

:::message
Router Cacheはクライアントサイドのキャッシュのため、上記によるキャッシュ破棄はあくまでユーザー端末内で起こります。**全ユーザーのRouter Cacheを同時に破棄する方法はありません**。

例えばあるユーザーがServer Actions経由で`revalidatePath()`を呼び出しても、他のユーザーはそれぞれ`staleTimes.static`(デフォルト: 5分)または`staleTimes.dynamic`(デフォルト: 0秒)経過後の画面更新までRouter Cacheを参照し続けます。
:::

## トレードオフ

### ドキュメントにはないRouter Cacheの挙動

Router Cacheの挙動はドキュメントにない挙動をすることも多く、非常に複雑です。特に筆者が注意しておくべき点として認識してるものを以下にあげます。

- ブラウザバック時は`staleTimes`の値にかかわらず、必ずRouter Cacheが利用される(キャッシュ破棄後であれば再取得する)
- `staleTimes.dynamic`に指定する値は、「キャッシュが保存された時刻」か「キャッシュが最後に利用された時刻」からの経過時間
- `staleTimes.static`に指定する値は、「キャッシュが保存された時刻」からの経過時間のみ

より詳細な挙動を知りたい方は、少々古いですが筆者の過去記事をご参照ください。

https://zenn.dev/akfm/articles/next-app-router-client-cache

---
title: "データ操作とServer Actions"
---

## 要約

データ操作はServer Actionsで実装することを基本としましょう。

## 背景

Pages Routerではデータ取得のために[`getServerSideProps()`↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)や[`getStaticProps()`↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-static-props)が提供されてましたが、データ操作アプローチは公式には提供されていませんでした。そのため、クライアントサイドを主体、または[API Routes↗︎](https://nextjs.org/docs/pages/building-your-application/routing/api-routes)を統合した3rd partyライブラリによるデータ操作の実装パターンが多く存在します。

- [SWR↗︎](https://swr.vercel.app/)
- [React Query↗︎](https://react-query.tanstack.com/)
- GraphQL
  - [Apollo Client↗︎](https://www.apollographql.com/docs/react/)
  - [Relay↗︎](https://relay.dev/)
- [tRPC↗︎](https://trpc.io/)
- etc...

しかし、API RouteはApp Routerにおいて[Route Handler↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/route)となり、定義の方法や参照できる情報などが変更されました。また、App Routerは多層のキャッシュを活用しているため、データ操作時にはキャッシュのrevalidate機能との統合が必要となるため、上記にあげたライブラリや実装パターンをApp Routerで利用するには多くの工夫や実装が必要となります。

## 設計・プラクティス

App Routerでのデータ操作は、従来からある実装パターンではなく[Server Actions↗︎](https://nextjs.org/docs/app/getting-started/updating-data)^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]を利用することが推奨されています。これにより、tRPCなどの3rd partyライブラリなどなしにクライアント・サーバーの境界を超えて関数を呼び出すことができ、データ変更処理を容易に実装できます。

:::message
Server Actionsはクライアント・サーバーの境界を超えて関数を呼び出しているように見えますが、実際には当然通信処理が伴うため、引数や戻り値には[Reactがserialize可能なもの↗︎](https://ja.react.dev/reference/rsc/use-server#serializable-parameters-and-return-values)のみを利用できます。

詳しくは[クライアントとサーバーのバンドル境界](part_2_bundle_boundary)を参照ください。
:::

```tsx :app/create-todo.ts
"use server";

export async function createTodo(formData: FormData) {
  // ...
}
```

```tsx :app/page.tsx
"use client";

import { createTodo } from "./create-todo";

export default function CreateTodo() {
  return (
    <form action={createTodo}>
      {/* ... */}
      <button>Create Todo</button>
    </form>
  );
}
```

上記の実装例では、サーバーサイドで実行される関数`createTodo()`をClient Components内の`<form action={createTodo}>`で渡しているのがわかります。このformを実際にsubmitすると、サーバーサイドで`createTodo()`が実行されます。

このように、非常にシンプルな実装でクライアントサイドからサーバーサイド関数を呼び出せることで、開発者はデータ操作の実装に集中できます。Server ActionsはReactの仕様ですが、実装はフレームワークに統合されているので、他にも以下のようなNext.jsならではのメリットが得られます。

### キャッシュのrevalidate

Next.jsは多層のキャッシュを活用しているため、データ操作時には関連するキャッシュのrevalidateが必要になります。Server Actions内で[`revalidatePath()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)や[`revalidateTag()`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)を呼び出すと、サーバーサイドの関連するキャッシュ([Data Cache↗︎](https://nextjs.org/docs/app/guides/caching#data-cache)や[Full Route Cache↗︎](https://nextjs.org/docs/app/guides/caching#full-route-cache))とクライアントサイドのキャッシュ([Router Cache↗︎](https://nextjs.org/docs/app/guides/caching#client-side-router-cache))がrevalidateされます。

```tsx :app/actions.ts
"use server";

export async function updateTodo() {
  // ...
  revalidateTag("todos");
}
```

:::message alert
Server Actionsで`revalidatePath()`/`revalidateTag()`もしくは`cookies.set()`/`cookies.delete()`を呼び出すと、Router Cacheが**全て**破棄され、呼び出したページのServer Componentsが再レンダリングされます。
必要以上に多用するとパフォーマンス劣化の原因になるので注意しましょう。
:::

### redirect時の通信効率

Next.jsではサーバーサイドで呼び出せる[`redirect()`↗︎](https://nextjs.org/docs/app/api-reference/functions/redirect)という関数があります。データ操作後にページをリダレイクトしたいことはよくあるユースケースですが、`redirect()`をServer Actions内で呼び出すとレスポンスにリダイレクト先ページの[RSC Payload↗︎](https://nextjs.org/docs/app/getting-started/server-and-client-components#on-the-server)が含まれるため、HTTPリダイレクトをせずに画面遷移できます。これにより、従来データ操作リクエストとリダイレクト後ページ情報のリクエストで2往復は必要だったhttp通信が、1度で済みます。

```tsx :app/actions.ts
"use server";

import { redirect } from "next/navigation";

export async function createTodo(formData: FormData) {
  console.log("create todo: ", formData.get("title"));

  redirect("/thanks");
}
```

上記のServer Actionsを実際に呼び出すと、遷移先の`/thanks`のRSC Payloadが含まれたレスポンスが返却されます。

```text
2:I[3099,[],""]
3:I[2506,[],""]
0:["lxbJ3SDwnGEl3RnM3bOJ4",[[["",{"children":["thanks",{"children":["__PAGE__",{}]}]},"$undefined","$undefined",true],["",{"children":["thanks",{"children":["__PAGE__",{},[["$L1",[["$","h1",null,{"children":"Thanks page."}],["$","p",null,{"children":"Thank you for submitting!"}]]],null],null]},["$","$L2",null,{"parallelRouterKey":"children","segmentPath":["children","thanks","children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L3",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","styles":null}],null]},[["$","html",null,{"lang":"en","children":["$","body",null,{"children":["$","$L2",null,{"parallelRouterKey":"children","segmentPath":["children"],"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L3",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":[["$","title",null,{"children":"404: This page could not be found."}],["$","div",null,{"style":{"fontFamily":"system-ui,\"Segoe UI\",Roboto,Helvetica,Arial,sans-serif,\"Apple Color Emoji\",\"Segoe UI Emoji\"","height":"100vh","textAlign":"center","display":"flex","flexDirection":"column","alignItems":"center","justifyContent":"center"},"children":["$","div",null,{"children":[["$","style",null,{"dangerouslySetInnerHTML":{"__html":"body{color:#000;background:#fff;margin:0}.next-error-h1{border-right:1px solid rgba(0,0,0,.3)}@media (prefers-color-scheme:dark){body{color:#fff;background:#000}.next-error-h1{border-right:1px solid rgba(255,255,255,.3)}}"}}],["$","h1",null,{"className":"next-error-h1","style":{"display":"inline-block","margin":"0 20px 0 0","padding":"0 23px 0 0","fontSize":24,"fontWeight":500,"verticalAlign":"top","lineHeight":"49px"},"children":"404"}],["$","div",null,{"style":{"display":"inline-block"},"children":["$","h2",null,{"style":{"fontSize":14,"fontWeight":400,"lineHeight":"49px","margin":0},"children":"This page could not be found."}]}]]}]}]],"notFoundStyles":[],"styles":null}]}]}],null],null],[null,"$L4"]]]]
4:[["$","meta","0",{"name":"viewport","content":"width=device-width, initial-scale=1"}],["$","meta","1",{"charSet":"utf-8"}]]
1:null
```

### JavaScript非動作時・未ロード時サポート

Next.jsのServer Actionsでは`<form>`の`action`propsにServer Actionsを渡すと、ユーザーがJavaScriptをOFFにしてたり、JavaScriptファイルが未ロードであっても動作します。

:::message
[公式ドキュメント↗︎](https://nextjs.org/docs/app/getting-started/updating-data#server-components)では「Progressive Enhancementのサポート」と記載されていますが、厳密にはJavaScript非動作環境のサポートとProgressive Enhancementは異なると筆者は理解しています。詳しくは以下をご参照ください。

https://developer.mozilla.org/ja/docs/Glossary/Progressive_Enhancement

:::

これにより、[FID↗︎](https://web.dev/articles/fid?hl=ja)(First Input Delay)の向上も見込めます。実際のアプリケーション開発においては、Formライブラリを利用しつつServer Actionsを利用するケースが多いと思われるので、筆者はJavaScript非動作時もサポートしてるFormライブラリの[Conform↗︎](https://conform.guide/)をおすすめします。

https://zenn.dev/akfm/articles/server-actions-with-conform

## トレードオフ

### サイト外で発生するデータ操作

Server Actionsは基本的にサイト内でのみ利用することが可能ですが、データ操作がサイト内でのみ発生するとは限りません。具体的にはヘッドレスCMSでのデータ更新など、サイト外でデータ操作が発生した場合にも、Next.jsで保持しているキャッシュをrevalidateする必要があります。

Route Handlerが`revalidatePath()`などを扱えるのはまさに上記のようなユースケースをフォローするためです。サイト外でデータ操作が行われた時には、Route Handlerで定義したAPIをWeb hookで呼び出すなどしてキャッシュをrevalidateしましょう。

:::message
Router Cacheはユーザー端末のインメモリに保存されており、全ユーザーのRouter Cacheを一括で破棄する方法はありません。上記の方法で破棄できるのは、サーバー側キャッシュのData CacheとFull Route Cacheのみです。
:::

### ブラウザバックにおけるスクロール位置の喪失

Next.jsにおけるブラウザバックではRouter Cacheが利用されます。この際には画面は即時に描画され、スクロール位置も正しく復元されます。

しかし、Server Actionsで`revalidatePath()`などを呼び出すなどすると、Router Cacheが破棄されます。Router Cacheがない状態でブラウザバックを行うと即座に画面を更新できないため、スクロール位置がうまく復元されないことがあります。

### Server Actionsの呼び出しは直列化される

Server Actionsは直列に実行されるよう設計されているため、同時に実行できるのは一つだけとなります。

https://quramy.medium.com/server-actions-%E3%81%AE%E5%90%8C%E6%99%82%E5%AE%9F%E8%A1%8C%E5%88%B6%E5%BE%A1%E3%81%A8%E7%94%BB%E9%9D%A2%E3%81%AE%E7%8A%B6%E6%85%8B%E6%9B%B4%E6%96%B0-35acf5d825ca

本書や公式ドキュメントで扱ってるような利用想定の限りではこれが問題になることは少ないと考えられますが、Server Actionsを高頻度で呼び出すような実装では問題になることがあるかもしれません。

そういった場合は、そもそも高頻度にServer Actionsを呼び出すような設計・実装を見直しましょう。

---
title: "第4部 レンダリング"
---

従来Pages Routerは[SSR↗︎](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG↗︎](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)という3つのレンダリングモデルをサポートしてきました。App Routerは引き続きこれらをサポートしていますが、これらに加え[Streaming SSR↗︎](https://nextjs.org/docs/app/getting-started/linking-and-navigating#streaming)に対応している点が大きく異なります。

第4部ではReactやNext.jsにおけるレンダリングの考え方について解説します。

---
title: "Server Componentsの純粋性"
---

## 要約

Reactコンポーネントのレンダリングは純粋であるべきです。Server Componentsにおいてもこれは同様で、データフェッチをメモ化することで純粋性を保つことができます。

## 背景

Reactは従来より、コンポーネントが**純粋**であることを重視してきました。Reactの最大の特徴の1つである[宣言的UI↗︎](https://ja.react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)も、コンポーネントが純粋であることを前提としています。

とはいえ、WebのUI実装には様々な副作用^[ここでは、他コンポーネントのレンダリングに影響しうる論理的状態の変更を指します]がつきものです。Client Componentsでは、副作用を`useState()`や`useEffect()`などのhooksに分離することで、コンポーネントの純粋性を保てるように設計されています。

https://ja.react.dev/learn/keeping-components-pure#side-effects-unintended-consequences

### 並行レンダリング

React18で並行レンダリングの機能が導入されましたが、これはコンポーネントが純粋であることを前提としています。

https://ja.react.dev/blog/2022/03/29/react-v18#what-is-concurrent-react

もし副作用が含まれるレンダリングを並行にしてしまうと、処理結果が不安定になりますが、副作用を含まなければレンダリングを並行にしても結果は安定します。このように、従来よりReactの多くの機能は、コンポーネントが副作用を持たないことを前提としていました。

## 設計・プラクティス

RSCにおいても従来同様、**コンポーネントが純粋**であることは非常に重要です。Next.jsもこの原則に沿って、各種APIが設計されています。

### データフェッチの一貫性

[データフェッチ on Server Components](part_1_server_components)で述べたように、Next.jsにおけるデータフェッチはServer Componentsで行うことが推奨されます。本来、データフェッチは純粋性を損なう操作の典型です。

```tsx
async function getRandomTodo() {
  // リクエストごとにランダムなTodoを返すAPI
  const res = await fetch("https://dummyjson.com/todos/random");
  return (await res.json()) as Todo;
}
```

上記の`getRandomTodo()`は呼び出しごとに異なるTodoを返す可能性が高く、そもそもリクエストに失敗する可能性もあるため戻り値は不安定です。このようなデータフェッチを内部的に呼び出す関数は、同じ入力（引数）でも出力（戻り値）が異なる可能性があり、純粋ではありません。

Next.jsは[Request Memoization↗︎](https://nextjs.org/docs/app/guides/caching#request-memoization)によって入力に対する出力を冪等に保ち、データフェッチをサポートしながらもレンダリングの範囲ではコンポーネントの純粋性を保てるよう設計されています。

```tsx
export async function ComponentA() {
  const todo = await getRandomTodo();
  return <>...</>;
}

export async function ComponentB() {
  const todo = await getRandomTodo();
  return <>...</>;
}

// ...

export default function Page() {
  // ランダムなTodoは1度だけ取得され、
  // 結果<ComponentA>と<ComponentB>は同じTodoを表示する
  return (
    <>
      <ComponentA />
      <ComponentB />
    </>
  );
}
```

### `cache()`によるメモ化

Request Memoizationは`fetch()`を拡張することで実現されていますが、DBアクセスなど`fetch()`を利用しないデータフェッチについても同様に純粋性は重要です。これは`React.cache()`を利用することで簡単に実装することができます。

```ts
export const getPost = cache(async (id: number) => {
  const post = await db.query.posts.findFirst({
    where: eq(posts.id, id),
  });

  if (!post) throw new NotFoundError("Post not found");
  return post;
});
```

ページを通して1回だけ発生するデータフェッチに対してなら、上記のように`cache()`でメモ化する必要はないように感じられるかもしれませんが、筆者は基本的にメモ化しておくことを推奨します。あえてメモ化したくない場合には、後々の改修で複数回呼び出すことになった際に一貫性が破綻するリスクが伴います。

### Cookie操作

Next.jsにおけるCookie操作も典型的な副作用の1つであり、Server Componentsからは変更操作であるCookieの`.set()`や`.delete()`は呼び出すことができません。

https://nextjs.org/docs/app/api-reference/functions/cookies

[データ操作とServer Actions](part_3_data_mutation)でも述べたように、Cookie操作や、APIに対するデータ変更リクエストなど変更操作はServer Actionsで行いましょう。

## トレードオフ

### Request Memoizationのオプトアウト

Request Memoizationは`fetch()`を拡張することで実現しています。`fetch()`の拡張をやめるようなオプトアウト手段は現状ありません。ただし、`fetch()`に渡す引数次第でRequest Memoizationをオプトアプトして都度データフェッチすることは可能です。

```ts
// クエリー文字列にランダムな値を付与する
fetch(`https://dummyjson.com/todos/random?_hash=${Math.random()}`);
// `signal`を指定する
const controller = new AbortController();
const signal = controller.signal;
fetch(`https://dummyjson.com/todos/random`, { signal });
```

---
title: "SuspenseとStreaming"
---

## 要約

Dynamic Renderingで特に重いコンポーネントのレンダリングは`<Suspense>`で遅延させて、Streaming SSRにしましょう。

## 背景

[Dynamic Rendering↗︎](https://nextjs.org/docs/app/guides/caching#static-and-dynamic-rendering)ではRoute全体をレンダリングするため、[Dynamic RenderingとData Cache](part_3_dynamic_rendering_data_cache)ではData Cacheを活用することを検討すべきであるということを述べました。しかし、キャッシュできないようなデータフェッチに限って無視できないほど遅いということはよくあります。

## 設計・プラクティス

Next.jsでは[Streaming SSR↗︎](https://nextjs.org/docs/app/getting-started/linking-and-navigating#streaming)をサポートしているので、重いデータフェッチを伴うServer Componentsのレンダリングを遅延させ、ユーザーにいち早くレスポンスを返し始めることができます。具体的には、Next.jsは`<Suspense>`のfallbackを元に即座にレスポンスを送信し始め、その後、`<Suspense>`内のレンダリングが完了次第結果がクライアントへと続いて送信されます。

[並行データフェッチ](part_1_concurrent_fetch)で述べたようなデータフェッチ単位で分割されたコンポーネント設計ならば、`<Suspense>`境界を追加するのみで容易に実装できるはずです。

### 実装例

少々極端な例ですが、以下のような3秒の重い処理を伴う`<LazyComponent>`は`<Suspense>`によってレンダリングが遅延されるので、ユーザーは3秒を待たずにすぐにページのタイトルや`<Clock>`を見ることができます。

```tsx
import { setTimeout } from "node:timers/promises";
import { Suspense } from "react";
import { Clock } from "./clock";

export const dynamic = "force-dynamic";

export default function Page() {
  return (
    <div>
      <h1>Streaming SSR</h1>
      <Clock />
      <Suspense fallback={<>loading...</>}>
        <LazyComponent />
      </Suspense>
    </div>
  );
}

async function LazyComponent() {
  await setTimeout(3000);

  return <p>Lazy Component</p>;
}
```

_即座にレスポンスが表示される_
![Streaming SSR loading](/images/nextjs-basic-principle/streaming-ssr-loading.png)

_3秒後、遅延されたレンダリングが表示される_
![Streaming SSR rendered](/images/nextjs-basic-principle/streaming-ssr-rendered.png)

## トレードオフ

### fallbackのLayout Shift

Streaming SSRを活用するとユーザーに即座に画面を表示し始めることができますが、画面の一部にfallbackを表示しそれが後に置き換えられるため、いわゆる**Layout Shift**が発生する可能性があります。

置き換え後の高さが固定ならば、fallbackも同様の高さで固定することでLayout Shiftを防ぐことができます。一方、置き換え後の高さが固定でない場合にはLayout Shiftが発生することになり、[Time to First Byte↗︎](https://web.dev/articles/ttfb?hl=ja)(TTFB)と[CumulativeLayout Shift↗︎](https://web.dev/articles/cls?hl=ja)(CLS)のトレードオフが発生します。

そのため実際のユースケースにおいては、コンポーネントがどの程度遅いのかによってレンダリングを遅延させるべきかどうか判断が変わってきます。筆者の感覚論ですが、たとえば200ms程度のデータフェッチを伴うServer ComponentsならTTFBを短縮するよりLayout Shiftのデメリットの方が大きいと判断することが多いでしょう。1sを超えてくるようなServer Componentsなら迷わず遅延することを選びます。

TTFBとCLSどちらを優先すべきかはケースバイケースなので、状況に応じて最適な設計を検討しましょう。

### StreamingとSEO

SEOにおける古い見解では、以下のようなものがありました。

- 「Googleに評価されたいコンテンツはHTMLに含むべき」
- 「見えてないコンテンツは評価されない」

これらが事実なら、Streaming SSRはSEO的に不利ということになります。

しかし、Vercelが独自に行った大規模な調査によると、Streaming SSRした内容が**Googleに評価されないということはなかった**とのことです。

https://vercel.com/blog/how-google-handles-javascript-throughout-the-indexing-process

この調査によると、JS依存なコンテンツがindexingされる時間については以下のようになっています。

- 50%~: 10s
- 75%~: 26s
- 90%~: ~3h
- 95%~: ~6h
- 99%~: ~18h

Googleのindexingにある程度遅延がみられるものの、上記のindexing速度は十分に早いと言えるため、コンテンツの表示がJS依存なStreaming SSRがSEOに与える影響は少ない(もしくはほとんどない)と筆者は考えます。

---
title: "第5部 その他のプラクティス"
---

Next.jsではリクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照できなかったり、リクエストに対して認可チェックを挟むレイヤーが直接なかったり、いくつかの実装パターンにおいて他のフレームワークや従来のPages Routerとは全く異なる実装が必要になります。

第5部では、Next.jsでよく利用されるいくつかのプラクティスについて解説します。

---
title: "リクエストの参照とレスポンスの操作"
---

## 要約

Server ComponentsやServer Functionsでは他フレームワークにあるようなリクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照することはできません。代わりに必要な情報を参照するためのAPIが提供されています。

## 背景

Pages Routerなど従来のWebフレームーワークでは、リクエストオブジェクト(`req`)やレスポンスオブジェクト(`res`)を参照することで様々な情報にアクセスしたり、レスポンスをカスタマイズするような設計が広く使われてきました。

```tsx
export const getServerSideProps = (async ({ req, res }) => {
  // リクエスト情報から`sessionId`というcookie情報を取得
  const sessionId = req.cookies.sessionId;

  // レスポンスヘッダーに`Cache-Control`を設定
  res.setHeader(
    "Cache-Control",
    "public, s-maxage=10, stale-while-revalidate=59",
  );

  // ...

  return { props };
}) satisfies GetServerSideProps<Props>;
```

しかし、Server ComponentsやServer Functionsではこれらのオブジェクトを参照することはできません。

## 設計・プラクティス

Next.jsではリクエストやレスポンスオブジェクトを提供する代わりに、必要な情報を参照するためのAPIが提供されています。

:::message
Server Componentsでリクエスト時の情報を参照する関数の一部は[Dynamic APIs↗︎](https://nextjs.org/docs/app/guides/caching#dynamic-apis)と呼ばれ、これらを利用するとRoute全体が[Dynamic Rendering↗︎](https://nextjs.org/docs/app/guides/caching#static-and-dynamic-rendering)となります。
:::

:::message
Next.jsは内部処理の都合で特殊なエラーを`throw`することがあります。そのため、下記のAPIに対し`try {} catch {}`するとNext.jsの動作に影響する可能性があります。
詳しくは[`unstable_rethrow()`↗︎](https://nextjs.org/docs/app/api-reference/functions/unstable_rethrow)を参照ください。
:::

### URL情報の参照

#### `params` props

Dynamic RoutesのURLパスの情報は[`params` props↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/page#params-optional)で提供されます。以下は`/posts/[slug]`と`/posts/[slug]/comments/[commentId]`というルーティングがあった場合の`params`の例です。

| URL                        | `params` props                       |
| -------------------------- | ------------------------------------ |
| `/posts/hoge`              | `{ slug: "hoge" }`                   |
| `/posts/hoge/comments/111` | `{ slug: "hoge", commentId: "111" }` |

```tsx
export default async function Page({
  params,
}: {
  params: Promise<{
    slug: string;
    commentId: string;
  }>;
}) {
  const { slug, commentId } = await params;
  // ...
}
```

#### `useParams()`

[`useParams()`↗︎](https://nextjs.org/docs/app/api-reference/functions/use-params)は、Client ComponentsでURLパスに含まれるDynamic Params（e.g. `/posts/[slug]`の`[slug]`部分）を参照するためのhooksです。

```tsx
"use client";

import { useParams } from "next/navigation";

export default function ExampleClientComponent() {
  const params = useParams<{ tag: string; item: string }>();

  // Route: /shop/[tag]/[item]
  // URL  : /shop/shoes/nike-air-max-97
  console.log(params); // { tag: 'shoes', item: 'nike-air-max-97' }

  // ...
}
```

#### `searchParams` props

[`searchParams` props↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional)は、URLのクエリー文字列を参照するためのpropsです。`searchParams` propsでは、クエリー文字列のkey-value相当なオブジェクトが提供されます。

| URL                             | `searchParams` props             |
| ------------------------------- | -------------------------------- |
| `/products?id=1`                | `{ id: "1" }`                    |
| `/products?id=1&sort=recommend` | `{ id: "1", sort: "recommend" }` |
| `/products?id=1&id=2`           | `{ id: ["1", "2"] }`             |

```tsx
type SearchParamsValue = string | string[] | undefined;

export default async function Page({
  searchParams,
}: {
  searchParams: Promise<{
    sort?: SearchParamsValue;
    id?: SearchParamsValue;
  }>;
}) {
  const { sort, id } = await searchParams;
  // ...
}
```

#### `useSearchParams()`

[`useSearchParams()`↗︎](https://nextjs.org/docs/app/api-reference/functions/use-search-params)は、Client ComponentsでURLのクエリー文字列を参照するためのhooksです。

```tsx
"use client";

import { useSearchParams } from "next/navigation";

export default function SearchBar() {
  const searchParams = useSearchParams();
  const search = searchParams.get("search");

  // URL -> `/dashboard?search=my-project`
  console.log(search); // 'my-project'

  // ...
}
```

### ヘッダー情報の参照

#### `headers()`

[`headers()`↗︎](https://nextjs.org/docs/app/api-reference/functions/headers)は、リクエストヘッダーを参照するための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

```tsx
import { headers } from "next/headers";

export default async function Page() {
  const headersList = await headers();
  const referrer = headersList.get("referrer");

  return <div>Referrer: {referrer}</div>;
}
```

### クッキー情報の参照と変更

#### `cookies()`

[`cookies()`↗︎](https://nextjs.org/docs/app/api-reference/functions/cookies)は、Cookie情報の参照や変更を担うオブジェクトを取得するための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

:::message
Cookieの`.set()`や`.delete()`といった操作は、Server ActionsやRoute Handlerでのみ利用でき、Server Componentsでは利用できません。詳しくは[Server Componentsの純粋性](part_4_pure_server_components)を参照ください。
:::

```tsx :app/page.tsx
import { cookies } from "next/headers";

export default async function Page() {
  const cookieStore = await cookies();
  const theme = cookieStore.get("theme");
  return "...";
}
```

```ts :app/actions.ts
"use server";

import { cookies } from "next/headers";

async function create(data) {
  const cookieStore = await cookies();
  cookieStore.set("name", "lee");

  // ...
}
```

### レスポンスのステータスコード

Next.jsはStreamingをサポートしているため、確実にHTTPステータスコードを設定する手段がありません。その代わりに、`notFound()`や`redirect()`といった関数でブラウザに対してエラーやリダイレクトを示すことができます。

これらを呼び出した際には、まだHTTPレスポンスの送信がまだ開始されていなければステータスコードが設定され、すでにクライアントにステータスコードが送信されていた場合には`<meta>`タグが挿入されてブラウザにこれらの情報が伝えられます。

#### `notFound()`

[`notFound()`↗︎](https://nextjs.org/docs/app/api-reference/functions/not-found)は、ページが存在しないことをブラウザに示すための関数です。Server Componentsで利用することができます。この関数が呼ばれた際には、該当Routeの`not-found.tsx`が表示されます。

```tsx
import { notFound } from "next/navigation";

// ...

export default async function Profile({ params }: { params: { id: string } }) {
  const user = await fetchUser(params.id);

  if (!user) {
    notFound();
  }

  // ...
}
```

#### `redirect()`

[`redirect()`↗︎](https://nextjs.org/docs/app/api-reference/functions/redirect)は、リダイレクトを行うための関数です。この関数はServer ComponentsやClient Components、Server FunctionsやRouter Handlerなど、多くの場所で利用できます。

```tsx
import { redirect } from "next/navigation";

// ...

export default async function Profile({ params }: { params: { id: string } }) {
  const team = await fetchTeam(params.id);
  if (!team) {
    redirect("/login");
  }

  // ...
}
```

#### `permanentRedirect()`

[`permanentRedirect()`↗︎](https://nextjs.org/docs/app/api-reference/functions/permanentRedirect)は、永続的なリダイレクトを行うための関数です。この関数はServer ComponentsやClient Components、Server FunctionsやRouter Handlerなど、多くの場所で利用できます。

```tsx
import { permanentRedirect } from "next/navigation";

// ...

export default async function Profile({ params }: { params: { id: string } }) {
  const team = await fetchTeam(params.id);
  if (!team) {
    permanentRedirect("/login");
  }

  // ...
}
```

### `unauthorized()`

[`unauthorized()`↗︎](https://nextjs.org/docs/app/api-reference/functions/unauthorized)は認証エラーを示すための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

:::message
このAPIは執筆時現在、実験的機能です。
:::

```tsx
import { verifySession } from "@/app/lib/dal";
import { unauthorized } from "next/navigation";

export default async function DashboardPage() {
  const session = await verifySession();
  if (!session) {
    unauthorized();
  }

  // ...
}
```

### `forbidden()`

[`forbidden()`↗︎](https://nextjs.org/docs/app/api-reference/functions/forbidden)は認可エラーを示すための関数です。この関数はServer Componentsなどのサーバー側処理でのみ利用することができます。

:::message
このAPIは執筆時現在、実験的機能です。
:::

```tsx
import { verifySession } from "@/app/lib/dal";
import { forbidden } from "next/navigation";

export default async function AdminPage() {
  const session = await verifySession();
  if (session.role !== "admin") {
    forbidden();
  }

  // ...
}
```

### その他

筆者が主要なAPIとして認識してるものは上記に列挙しましたが、Next.jsでは他にも必要に応じて様々なAPIが提供されています。上記にないユースケースで困った場合には、公式ドキュメントより検索してみましょう。

https://nextjs.org/docs

## トレードオフ

### `req`拡張によるセッション情報の持ち運び

従来`req`オブジェクトは、3rd partyライブラリが拡張して`req.session`にセッション情報を格納するような実装がよく見られました。Next.jsではこのような実装はできず、これに代わるセッション管理の仕組みなどを実装する必要があります。

以下は、GitHub OAuthアプリとして実装したサンプル実装の一部です。`sessionStore.get()`でRedisに格納したセッション情報を取得できます。

https://github.com/AkifumiSato/nextjs-book-oauth-app-example/blob/main/app/api/github/callback/route.ts#L12

セッション管理の実装が必要な方は、必要に応じて上記のリポジトリを参考にしてみてください。

---
title: "認証と認可"
---

## 要約

アプリケーションで認証状態を保持する代表的な方法としては以下2つが挙げられ、Next.jsにおいてもこれらを実装することが可能です。

- 保持したい情報をCookieに保持（JWTは必須）
- セッションとしてRedisなどに保持（JWTは任意）

また、代表的な認可チェックには以下2つが考えられます。

- URL認可
- データアクセス認可

これらの認可チェックは両立が可能ですが、URL認可の実装にはNext.jsならではの制約が伴います。

## 背景

Webアプリケーションにおいて、認証と認可は非常にありふれた一般的な要件です。

:::message
認証と認可は混在されがちですが、別物です。これらの違いについて自信がない方は、筆者の[過去記事↗︎](https://zenn.dev/akfm/articles/authentication-with-security)を参照ください。
:::

しかし、Next.jsにおける認証認可の実装には、従来のWebフレームワークとは異なる独自の制約が伴います。

これはNext.jsが、RSCという**自律分散性**と**並行実行性**を重視したアーキテクチャ^[参考: [Reactチームが見てる世界、Reactユーザーが見てる世界↗︎](https://zenn.dev/akfm/articles/react-team-vision)]に基づいて構築されていることや、edgeランタイムとNode.jsランタイムなど**多層の実行環境**を持つといった、従来のWebフレームワークとは異なる特徴を持つことに起因します。

### 並行レンダリングされるページとレイアウト

Next.jsでは、Route間で共通となる[レイアウト↗︎](https://nextjs.org/docs/app/getting-started/layouts-and-pages)を`layout.tsx`などで定義することができます。特定のRoute配下（e.g. `/dashboard`配下など）に対する認可チェックをレイアウト層で一律実装できるのでは、と考える方もいらっしゃると思います。しかし、このような実装は一見期待通りに動いてるように見えても、RSC Payloadなどを通じて情報漏洩などにつながるリスクがあり、避けるべき実装です。

これは、Next.jsの並行実行性に起因する制約です。Next.jsにおいてページとレイアウトは並行にレンダリングされるため、必ずしもレイアウト層に実装した認可チェックがページより先に実行されるとは限りません。そのため、ページ側で認可チェックをしていないと予期せぬデータ漏洩が起きる可能性があります。

これらの詳細な解説については以下の記事が参考になります。

https://zenn.dev/moozaru/articles/0d6c4596425da9

### Server ComponentsでCookie操作は行えない

RSCではデータ取得をServer Components、データ変更をServer Actionsという責務分けがされています。[Server Componentsの純粋性](part_4_pure_server_components)でも述べたように、Server Componentsにおける並行レンダリングやRequest Memoizationは、レンダリングに副作用が含まれないという前提の元に設計されています。

Cookie操作は他のコンポーネントのレンダリングに影響する可能性がある副作用です。そのため、Next.jsにおけるCookie操作である`.set()`や`.delete()`は、Server ActionsかRoute Handler内でのみ行うことができます。

### 制限を伴うmiddleware

Next.jsのmiddlewareはユーザーからのリクエストに対して一律処理を差し込むことができますが、v15.4まではランタイムがedgeに限定されており、Node.js APIが利用できなかったりDB操作系が非推奨など様々な制限が伴います。

:::message
[Next.js v15.5↗︎](https://nextjs.org/blog/next-15-5#nodejs-middleware-stable)でmiddlewareのNode.jsランタイムがStableとなりました。
:::

## 設計・プラクティス

Next.jsにおける認証認可は、上述の制約を踏まえて実装する必要があります。考えるべきポイントは大きく以下の3つです。

- [#認証状態の保持](#認証状態の保持)
- [#URL認可](#URL認可)
- [#データアクセス認可](#データアクセス認可)

### 認証状態の保持

サーバー側で認証状態を参照したい場合、Cookieを利用することが一般的です。認証状態をJWTにして直接Cookieに格納するか、もしくはRedisなどにセッション状態を保持してCookieにはセッションIDを格納するなどの方法が考えられます。

公式ドキュメントに詳細な解説があるので、本書でこれらの詳細は割愛します。

https://nextjs.org/docs/app/guides/authentication#session-management

筆者は、拡張されたOAuthやOIDCを用いることが多く、セッションIDをJWTにしてCookieに格納し、セッション自体はRedisに保持する方法をよく利用します。こうすることで、アクセストークンやIDトークンをブラウザ側に送信せずに済み、セキュリティ性の向上やCookieのサイズを節約などのメリットが得られます。また、Cookieに格納するセッションIDはJWTにすることで、改竄を防止することができます。

:::details GitHub OAuthアプリのサンプル実装
以下はGitHub OAuthアプリとして実装したサンプル実装の一部です。GitHubからリダイレクト後、CSRF攻撃対策のためのstateトークン検証、アクセストークンの取得、セッション保持などを行っています。

https://github.com/AkifumiSato/nextjs-book-oauth-app-example/blob/main/app/api/github/callback/route.ts
:::

### URL認可

URL認可の実装は多くの場合、認証状態や認証情報に基づいて行われます。認可処理を`verifySession()`として共通化した場合、各ページで以下のような実装を行うことになるでしょう。

```tsx :page.tsx
export default async function Page() {
  await verifySession(); // 認可に失敗したら`/login`にリダイレクト

  // ...
}
```

CookieにJWTを格納している場合は、middlewareでJWTの検証を行うことができます。認証状態をJWTに含めている場合は、さらに細かいチェックも可能です。一方、セッションIDのみをJWTに含めるようにしている場合には、IDの有効性やセッション情報の取得にRedisやDB接続が必要になるため、edgeランタイムのmiddlewareで行えるのは[楽観的チェック↗︎](https://nextjs.org/docs/app/guides/authentication#optimistic-checks-with-middleware-optional)に留まるということに注意しましょう。

### データアクセス認可

[Request Memoization](part_1_request_memoization)で述べたように、Next.jsではデータフェッチ層を分離して実装するケースが多々あります。データアクセス認可が必要な場合は、分離したデータフェッチ層で実装しましょう。

例えばVercelのようなSaaSにおいて、有償プランユーザーのみが利用可能なデータアクセスがあった場合、データフェッチ層に以下のような認可チェックを実装すべきでしょう。

```ts
// 🚨`unauthorized()`はv15時点でExperimental
import { unauthorized } from "next/navigation";

export async function fetchPaidOnlyData() {
  if (!(await isPaidUser())) {
    unauthorized();
  }

  // ...
}
```

X（旧Twitter）のようにブロックやミュートなど、きめ細かいアクセス制御（Fine-Grained Access Control）が必要な場合は、バックエンドAPIにアクセス制御を隠蔽する場合もあります。

```ts
// 🚨`forbidden()`はv15時点でExperimental
import { forbidden } from "next/navigation";

export async function fetchPost(postId: string) {
  const res = await fetch(`https://dummyjson.com/posts/${postId}`);
  if (res.status === 401) {
    forbidden();
  }

  return (await res.json()) as Post;
}
```

:::message
[公式ドキュメント↗︎](https://nextjs.org/docs/app/guides/authentication#creating-a-data-access-layer-dal)やVercelの[SaaS参考実装↗︎](https://github.com/vercel/nextjs-subscription-payments)では、認可エラーで`redirect("/login")`のようにリダイレクトするのみのものが多いですが、認可エラー=必ずしもリダイレクトではありません。
:::

:::message
Next.jsはStreamingをサポートしているため、確実にHTTPステータスコードを設定する手段がありません。そのため、上記参考実装の`unauthorized()`や`forbidden()`利用時もHTTPステータスコードが200になる可能性があります。
詳しくは[リクエストの参照とレスポンスの操作](part_5_request_ref)を参照ください。
:::

## トレードオフ

### URL認可の冗長な実装

認証状態に基づくURL認可はありふれた要件ですが、認証状態を確認するのにRedisやDBのデータ参照が必要な場合、edgeランタイムのmiddlewareでは行えません。そのため、前述のように各`page.tsx`で認可チェックを行う必要があり、冗長な実装になります。

```tsx :page.tsx
export default async function Page() {
  await verifySession(); // 認可に失敗したら`/login`にリダイレクト

  // ...
}
```

---
title: "エラーハンドリング"
---

## 要約

Next.jsにおけるエラーは主に、Server ComponentsとServer Functionsの2つで発生します。

Server Componentsのエラーは、エラー時のUIを`error.tsx`や`not-found.tsx`で定義します。一方Server Functionsにおけるエラーは、基本的に戻り値で表現することが推奨されます。

## 背景

Next.jsにおけるエラーは、クライアントかサーバーか、データ参照か変更かで分けて考える必要があり、具体的には以下の3つを分けて考えることになります。

- Client Components
- Server Components
- Server Functions

特に、Server ComponentsとServer Functionsは外部データに対する操作を伴うことが多いため、エラーが発生する可能性が高くハンドリングが重要になります。

### クライアントサイドにおけるレンダリングエラー

後述のNext.jsが提供するエラーハンドリングは、サーバーサイドで発生したエラーにのみ対応しています。クライアントサイドにおけるレンダリングエラーのハンドリングが必要な場合は、開発者が自身で`<ErrorBoundary>`を定義する必要があります。

https://ja.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary

また、クライアントサイドにおいてレンダリングエラーが発生する場合、ブラウザの種類やバージョンなどに依存するエラーの可能性が高く、サーバーサイドとは異なりリトライしても回復しない可能性があります。クライアントサイドのエラーは当然ながらサーバーに記録されないので、開発者はエラーが起きた事実さえ把握が難しくなります。

クライアントサイドのエラー実態を把握したい場合、Datadogなどの[RUM↗︎](https://www.datadoghq.com/ja/product/real-user-monitoring/)（Real User Monitoring）導入も同時に検討しましょう。

## 設計・プラクティス

Next.jsにおけるエラーは主に、Server ComponentsとServer Functionsの2つで考える必要があります。

### Server Componentsのエラー

Next.jsでは、Server Componentsの実行中にエラーが発生した時のUIを、Route Segment単位の`error.tsx`で定義することができます。Route Segment単位なのでレイアウトはそのまま、`page.tsx`部分に`error.tsx`で定義したUIが表示されます。以下は[公式ドキュメント↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/error#how-errorjs-works)にある図です。

![エラー時のUIイメージ](/images/nextjs-basic-principle/error-ui.png)

`error.tsx`は主にServer Componentsでエラーが発生した場合に利用されます。

:::message
厳密にはSSR時のClient Componentsでエラーが起きた場合にも`error.tsx`が利用されます。
:::

`error.tsx`はClient Componentsである必要があり、propsで`reset()`を受け取ります。`reset()`は、再度ページのレンダリングを試みるリロード的な振る舞いをします。

```tsx
"use client";

import { useEffect } from "react";

export default function ErrorPage({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    console.error(error);
  }, [error]);

  return (
    <div>
      <h2>Something went wrong!</h2>
      <button type="button" onClick={() => reset()}>
        Try again
      </button>
    </div>
  );
}
```

#### Not Foundエラー

HTTPにおける404 Not FoundエラーはSEO影響もあるため、その他のエラーとは特に区別されることが多いエラーです。Next.jsでは404相当のエラーをthrowするためのAPIとして`notFound()`を提供しており、[`notFound()`↗︎](https://nextjs.org/docs/app/api-reference/functions/not-found)が呼び出された際のUIは[`not-found.tsx`↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/not-found)で定義できます。

:::message
多くの場合`notFound()`を呼び出すとHTTPステータスコードとして404 Not Foundが返されますが、`<Suspense>`内などで`notFound()`を利用すると200 OKが返されることがあります。この際Next.jsは、`<meta name="robots" content="noindex" />`タグを挿入してGoogleクローラなどに対してIndexingの必要がないことを示します。
:::

:::message
Next.jsには[`unauthorized()`↗︎](https://nextjs.org/docs/app/api-reference/functions/unauthorized)や[`forbidden()`↗︎](https://nextjs.org/docs/app/api-reference/functions/forbidden)も提供されていますが、執筆時現在これらは実験的機能となっています。今後変更される可能性もあるので注意しましょう。
:::

### Server Functionsのエラー

Server Functionsのエラーは、**予測可能なエラー**と**予測不能なエラー**で分けて考える必要があります。

Server Functionsは多くの場合、Server Actions^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]としてデータ更新の際に呼び出されます。何かしらの理由でデータ更新に失敗したとしても、ユーザーは再度更新をリクエストできることが望ましいUXと考えられます。しかし、Server Actionsではエラーが`throw`されると、前述の通り`error.tsx`で定義したエラー時のUIが表示されます。`error.tsx`が表示されることで、直前までページで入力していた`<form>`の入力内容などが失われると、ユーザーは操作を最初からやり直すことになりかねません。

そのため、Server Functionsにおける予測可能なエラーは`throw`ではなく、**戻り値でエラーを表現**することが推奨されます。予測不能なエラーに対しては当然ながら対策できないので、予測不能なエラーが発生した場合は`error.tsx`が表示されることは念頭に置いておきましょう。

以下は[conform↗︎](https://ja.conform.guide/integration/nextjs)を使ったServer Actionsにおけるzodバリデーションの実装例です。バリデーションエラー時は`throw`せず、`submission.reply()`を返している点がポイントです。

```tsx
"use server";

import { redirect } from "next/navigation";
import { parseWithZod } from "@conform-to/zod";
import { loginSchema } from "@/app/schema";

export async function login(prevState: unknown, formData: FormData) {
  const submission = parseWithZod(formData, {
    schema: loginSchema,
  });

  if (submission.status !== "success") {
    return submission.reply();
  }

  // ...

  redirect("/dashboard");
}
```

formライブラリを利用してない場合は、以下のように自身で戻り値を定義しましょう。

```tsx
"use server";

import { redirect } from "next/navigation";

export async function login(prevState: unknown, formData: FormData) {
  const submission = parseWithZod(formData, {
    schema: loginSchema,
  });

  if (formData.get("email") !== "") {
    return { message: "メールアドレスは必須です。" };
  }

  // ...

  redirect("/dashboard");
}
```

## トレードオフ

特になし
