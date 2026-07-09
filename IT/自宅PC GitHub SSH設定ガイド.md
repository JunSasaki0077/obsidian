# 自宅PC GitHub SSH 設定ガイド

会社PC（Work PC）と同じ個人アカウント用。所要時間 約10分。

---

## Step 1: SSH 鍵ペアを作成する

以下のコマンドを実行する。メールアドレスは自分のものに変える。

```bash
ssh-keygen -t ed25519 -C "あなたのメールアドレス"
```

実行するといくつか質問される：

```
# 保存先を聞かれる → そのままEnterでOK
Enter file in which to save the key: ~/.ssh/id_ed25519

# パスフレーズを設定する（推奨）
Enter passphrase: （任意のパスワードを入力）
Enter same passphrase again: （同じパスワードをもう一度）
```

> [!warning] パスフレーズは忘れずに
> 忘れると鍵を作り直すしかない。パスワードマネージャー等に保存しておくこと。

---

## Step 2: SSH config を設定する

`~/.ssh/config` に以下を追記する（ファイルがなければ新規作成）。

```
# 個人アカウント用
Host github.com-personal
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_ed25519
```

> [!tip] Port 443 について
> 会社ネットワーク等でポート22が塞がれていても443なら通ることが多いため、最初からこの設定にしておくのが安心。

---

## Step 3: SSH エージェントに登録する

`--apple-use-keychain` オプションで Mac のキーチェーンに保存され、再起動後も自動で読み込まれる。

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

パスフレーズを聞かれたら Step 1 で設定したものを入力。`Identity added` と表示されれば成功。

---

## Step 4: GitHub に公開鍵を登録する

まず公開鍵の中身を表示してコピーする。

```bash
cat ~/.ssh/id_ed25519.pub
```

`ssh-ed25519 AAAA...` から始まる1行が表示される。**最後のメールアドレスも含めて全部**コピーする。

GitHub にアクセスして登録：

```
Settings → SSH and GPG keys → New SSH key

Title    : Home PC
Key type : Authentication Key
Key      : （コピーした内容をペースト）
```

> [!warning] Work PC の鍵は削除しない
> GitHub には複数の鍵を登録できる。会社PCの「Work PC」はそのまま残して「Home PC」を追加する形にすること。

---

## Step 5: 接続を確認する

```bash
ssh -T git@github.com-personal
```

初回は以下の確認が出る。`yes` と入力する。

```
The authenticity of host '[ssh.github.com]:443' can't be established.
Are you sure you want to continue connecting? yes
```

`Hi JunSasaki0077!` と表示されれば完了。あとは `git push` するだけ。

---

## よくあるエラーと対処法

### `Permission denied (publickey).`
- **原因**: 公開鍵が GitHub に登録されていない、またはエージェントに鍵が読み込まれていない
- **対処**: Step 3 の `ssh-add --apple-use-keychain` を再実行 → Step 4 で GitHub に鍵が登録されているか確認

### `nc: authentication method negotiation failed`
- **原因**: ポート22が塞がれている（ネットワーク制限）
- **対処**: Step 2 の config で `Port 443` と `HostName ssh.github.com` が設定されているか確認

### `fatal: not a git repository`
- **原因**: git push をリポジトリ以外のディレクトリで実行している
- **対処**: `cd ~/Documents/GitHub/obsidian` などでリポジトリのディレクトリに移動してから実行

### `Enter passphrase for ~/.ssh/id_ed25519:`
- **原因**: エージェントに鍵が読み込まれていない（再起動後など）
- **対処**: `ssh-add --apple-use-keychain ~/.ssh/id_ed25519` を実行

### `zsh: bad pattern: [200~!`
- **原因**: コマンドをコピペしたとき制御文字が混入した
- **対処**: `!` を除いてコマンドを直接入力するか、ターミナルに直接貼り付け直す

---

## 気をつけるポイント

- パスフレーズは設定することを推奨。鍵ファイルが盗まれた場合の保険になる。必ずどこかに保存しておく
- 公開鍵（`.pub`）は GitHub に貼り付ける用。秘密鍵（`.pub` なし）は絶対に外部に渡さない
- GitHub の SSH keys 一覧で会社PCの「Work PC」が残っていることを確認してから「Home PC」を追加する
- リモートが `git@github.com-personal:...` の形式になっていることが前提。`git remote -v` で確認できる
