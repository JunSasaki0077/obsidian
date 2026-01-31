#IT #エラー

Pritterが上手く効かない場合は
.prettierrcファイルを作成し、以下のコードを貼り付ける

```ts
{

"tabWidth": 4,

"semi": true,

"singleQuote": true,

"jsxSingleQuote": true,

"arrowParens": "always",

"printWidth": 100

}
```

これでもだめなら、設定を確認する
⌘+ 、で設定を開き、検索欄で`format`と検索

`Defaut Format`の部分がPrettierになっているか確認