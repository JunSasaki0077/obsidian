#IT #知識 

Next.jsでコードを書いていたらとあるエラーに遭遇しました。

	Error: Text content does not matchserver-rendered HTML.  
	 Warning: Text content did not 
	 match.Server: {sever text} Client: {client text}

なぜこのようなエラー起きたのかと、解消方法について書きます。

## そもそもハイドレーションとは

ハイドレーションとは水分補給を意味しており、
静的なHTMLに対してJavaScriptという水分をあたえることにより動的になることを指します。

もう少し詳しく書くとSSRを実行する場合、
クライアント側でJacaScriptが全てのレンダリングを行うのではなく、
サーバー側で事前に静的なHTMLを生成してクライアントに送信します。
この場合表示速度は高速になりますが、現段階ではインタラクションができません
（クリックしても何も起きない等）

HTMLをレンダリングした後に必要なJavaScriptをサーバーから生成し、
生成されているHTMLに結びつけます。
これによりインタラクションができるようになる。（クリックできる！）

`静的なHTMLにJavaScriptを結びつけてインタラクティブにするプロセスをHydrationと呼ぶ`


## なぜエラーが起きるのか。

サーバー上で事前レンダリングされたJavaScriptとHTMLで最初にレンダリングされた
JavaScriptとの間に不一致がある場合にエラーが発生する。

#### 一般的な原因

 サーバー側でプリレンダリング(事前生成)された、HTMLに対して、
 クライアント側でレンダリングされたHTMLが違った際にエラーが発生します。

さらにハイドレーションエラーは次の場合に起こります。

1. HTMLタグのネストが正しくない
2. レンダリングロジックで`typeof window !== "undefined"` のようなチェックを行う
3. レンダリングロジックで`window`や`localStorage`などのブラウザ専用APIを使用する
4. レンダリングロジックで`Date()`コンストラクタなどの時間依存APIを使用する
5. ブラウザの拡張機能でHTMLを変更する
6. [CSS-in-JSライブラリの設定が間違っている](https://nextjs.org/docs/app/building-your-application/styling/css-in-js)
7.  [Cloudflare Auto Minify](https://developers.cloudflare.com/speed/optimization/content/auto-minify/)など、HTMLレスポンスを変更しようとする、正しく構成されていないEdge/CDN[](https://developers.cloudflare.com/speed/optimization/content/auto-minify/)

 #### エラーの解消策
1. `useEffect`を使用してクライントのみで実行する
2. 特定のコンポーネントでSSRを無効にする
3. `suppressHydrationWarning`をhtmlタグ、bodyタグに使用する

テーブル内で適切なtbodyタグを使用することによって解消されているケースもある
参照URL
「[NextでReact Hydrationエラーを解決する方法」](https://stackoverflow.com/questions/73451295/how-to-solve-react-hydration-error-in-next)

検証して解決できない場合は最後の手段として`suppressHydrationWarning`を
使用する。
書き方やバグではなく`仕組み上どうしても同じにならない値`があるから
例：
```tsx
<p>{Date.now()}</p>
```
＊時間なので毎回変わる

```tsx
<p>{Math.random()}</p>
```
＊値が毎回ランダムに生成される為