#IT #知識 

## ORMとは
#### 一言でいうと

`Object-Relational Mapping`の略
オブジェクト指向言語をデータベースの言葉に変換してくれるコンパイラー的な役割をしている。

#### どう違うのか

Rubyでデータベースからすべての
ユーザーを取得するコードを例に紹介します。
#### ORMを使用しない場合

```d
users = Array.new sql = "SELECT * FROM users" 
rows = some_sql_module.query(sql); 
rows.each do |row| user = User.new; 
 user.id = row[:id]
 user.name = row[:name] 
 user.email = row[:email] 
 users << user 
end
```

#### ORMを使用する場合

```ruby
users = User.all
```

#### 画像で比較

![img](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F3529304%2F8ce979fb-1bfb-4fec-1393-c556ef8dcc54.png?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&s=5a089b2707326f07785d4375ee299748)



## 利点

1. SQLを書かなくてよくなる
2. オブジェクト指向型言語 でコードを書ける