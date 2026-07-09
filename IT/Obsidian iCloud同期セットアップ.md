# Obsidian iCloud 同期セットアップ

Mac + iPhone を iCloud で同期する手順。会社PCは今のGit運用のまま。

---

## デバイス構成

| 端末 | 同期方法 |
|---|---|
| 会社Mac | Git（今のまま） |
| 自宅Mac | iCloud（+ 定期的にGit） |
| iPhone | iCloud |

---

## 自宅Macでの作業

### Step 1: iCloud Drive をオンにする

システム設定 → Apple ID → iCloud → **iCloud Drive をオン**

### Step 2: VaultをiCloud Driveに移動する

```bash
mkdir -p ~/Library/Mobile\ Documents/com~apple~CloudDocs/Obsidian
mv ~/Documents/GitHub/obsidian ~/Library/Mobile\ Documents/com~apple~CloudDocs/Obsidian/
```

### Step 3: Obsidianで開き直す

Obsidian を開いて「Open folder as vault」→ iCloud Drive 内の `obsidian` フォルダを選択

---

## iPhoneでの作業

1. Obsidian アプリをインストール
2. Vault 作成時に「**Store in iCloud**」をオン
3. iCloud 内の `obsidian` フォルダを選択

---

## 会社Macでの作業（今のまま）

変更なし。帰る前に `git push` するだけ。

```bash
cd ~/Documents/GitHub/obsidian
git add .
git commit -m "vault backup: $(date '+%Y-%m-%d %H:%M:%S')"
git push
```

---

## 注意点

- iCloud の無料枠は **5GB**。テキストのみなら余裕で収まる
- 同じファイルを2台で同時に編集すると競合が起きることがある。編集し終わったら少し待つ
- 会社PCの `.git` フォルダは iCloud に入れない（移動先のフォルダ外にあるのでそのまま OK）
- オフライン中の編集はオンライン復帰時に自動同期される

---

## よくあるエラーと対処法

### Vaultが見つからない（Obsidianで開けない）
- Finder で「iCloud Drive → Obsidian → obsidian」フォルダがあるか確認
- iCloud の同期が完了するまで数分かかることがある

### iPhoneにVaultが表示されない
- iPhone の設定 → Apple ID → iCloud → iCloud Drive がオンになっているか確認
- Obsidian アプリを一度終了して再起動する

### 競合ファイルが発生した（ファイル名に端末名が入った別ファイルができた）
- 2つのファイルを開いて内容を見比べ、新しい方に統合する
- 統合したら古い方のファイルを削除
