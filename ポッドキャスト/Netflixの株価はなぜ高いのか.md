

## 📻️基本情報

- 番組名:  #ゆるコンピュータ科学ラジオ
- 公開日: 2026-01-18
- 参考リンク:
	- [Netflix Open Connectに関する記事](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqa0t1c0x6ZUFTcDl2akIxMEVUcGhjYzhVSW1NUXxBQ3Jtc0ttUDhtS2VGQUJFZnNDYTJHbWZDYmliUlFhY19QRGlrSFRhZFhnektmZ01yMW9ISFJoRHlHT0dpaThkODFmOHB0VTQtUmNZV3hiXzhLUjVEZlJna25sRjRsbHNzb1hmZWlBVnlLbndIUU0wLVhfcmxXWQ&q=https%3A%2F%2Fopenconnect.netflix.com%2Fja_jp%2F&v=GA63OFcOakQ) 
	- [Netflixが世界中の下りの通信量のうち○%を占めていた話](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbEZReXNUcm10S0lFNmhZSnJ4eTVGNTl5QzBrZ3xBQ3Jtc0trdzdwVTZaSGhRM0I3Z29lZXN4bTUtTEx1LXJldmFrNFBIU0E4UkN2U0liRnNQOFh0M1otTFdjWGRyaWY2TjhMbTJCUFQzcUlTVzQtbjYzb0lPM0NNY0VTbXBHZWl3d3lDVlUwZGxxZF94Y2Y0Rlg2QQ&q=https%3A%2F%2Fwww.applogicnetworks.com%2Finthenews%2Fnetflix-eats-up-15-of-global-downstream-traffic&v=GA63OFcOakQ) 
	- [技術やカオスモンキーについて書かれた記事 ](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbV9MMUJRX0VvRnpRR01sMWp6VkV5cmFtOTZWZ3xBQ3Jtc0tuNjFBVnlrbWN1T3NPSUlsREdlbENlX2R5S3ZMc2R5TFQ2V1ZuQjJXWHEwc2ZSNkhpVGhOVWdwSmNPTDc0azFkWUU4RTNfeDlEU1lqTTFITXRuM1kzR1k2ci1YVzNHS2RJQXB0MXVCek5wNFJWcVZldw&q=https%3A%2F%2Ftalent500.com%2Fblog%2Fnetflix-devops-chaos-engineering-reliability%2F&v=GA63OFcOakQ) 
	- [Netflixのサーバーについて ](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbUZtdEdrS0ZhMVBTSzVJT0lYRTA5RThOT3g4UXxBQ3Jtc0trVWdpLWQzQ0Nfa0VlY1dSdGJCZk81VGwwM0ZUWWFoS1hidWlZZ3JoSTlLd3VhVFJieDM5dl9SRE5MUTJidHpyNjRkVGswRVppdFZVMzRDeHFZNXZhV2pDX0xncEpNdTN0VktoczNXbHVqdTNBejhUcw&q=https%3A%2F%2Fabout.netflix.com%2Fen%2Fnews%2Fopen-connect-celebrating-a-decade-of-smooth-and-efficient-streaming&v=GA63OFcOakQ)

## 重要ポイント

- 他の動画配信サイトはCDNレンタル会社のサーバーを借りて動画を配信しているが
  Netflixは独自でCDN(Contet Delivery Network)を構築し、世界中のいたるところにサーバーを置いているため
  一斉にアクセスがあってもサーバーが落ちることもないし、快適に動画を見ることができる
- Netflix会員数は約３億人で世界中の通信料うち約15％ぐらいを占めている
- 10年後はもしかしたら動画配信サイトがサーバーを貸している業容に変わる可能性がある
	AMAZONも元はただのECサイトだったが、クリスマス競売でサーバーが落ちてしまう為、
	大量に増設をしたけど、余っているサーバーを貸しますよというビジネスを始めた
	それが`AWS`であり。現在はECサイトより売上が高い
- 自社でCDNを持っているメリットとして`利益率`と`最適化`がある。これは他社が一朝一夕で真似することができない。

- `利益率`はポイントの1番最初に述べている通り、
  レンタル会社から借りてサイトを運用をするのだが
  これが、通信料１Gに対していくらと値段がかかってしまう。
  ましてや世界中の15%の通信料を占めているNetflixならなおさら

- `最適化`に関しては作品の人気予測に合わせて分散を最適化することができる。
  *レンタル映画をしているゲオで例えると人気作なら各所に置くけど
  マイナーな作品なら大きいゲオだけに置くみたいなこと*
- 他にも技術的優位性はあり、動画の圧縮独自規格がある(mp3みたいな)
## 自分の考え

- 他国の動画配信サービスで倒産してしまった会社とかをみかけたが
  倒産もしないし、Disneyプラスなど、他の強い動画配信サイトがあるにも関わらず
  TOPにいる理由がわかった。