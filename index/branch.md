#Git #IT #知識 

[[Git]]のコマンドの一つ
`git brench　ブランチ名`とすることで[[commit]]からの分岐を行える

branchがない場合

```mermaid
    gitGraph
       commit id: "A"
       commit id: "B"
       commit id: "C"

```
branchを作成しcommitをすると

```mermaid
    gitGraph
       commit id: "ボタン追加"
       commit id: "テキスト変更"
       commit id: "Header追加"
       branch feature/login
       commit id: "ログインページ作成"

```
上記のように枝分かれするようになります