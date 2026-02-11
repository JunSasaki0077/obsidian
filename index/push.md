#Git #IT #知識 

[[Git]]コマンドの一つ
今いるブランチの最新[[commit]]を[[GitHub]]に送るために使用する

## pushができないとき

```nginx
rejected
```
と表示された際は
- 誰かが先にpushしている
- 自分の履歴が古い

その場合
```nginx
git pull
```
git [[pullを