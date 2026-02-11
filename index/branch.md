#Git #IT #知識 

[[Git]]のコマンドの一つ
`git brench　ブランチ名`とすることで現在の[[commit]]**からの分岐を行えることができ**
作業を分けることができる。

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
mainは本番環境の為なるべく触らないようにし、
branchを切って作業を行うことになります。
切ったbranchで作業を行ったあと[[push]]して
[[GitHub]]上で[[プルリクエスト]]を行い
レビューを受けて問題なければ[[marge]]を行う
