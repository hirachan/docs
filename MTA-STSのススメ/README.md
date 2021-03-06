# MTA-STSのススメ

## MTA-STSとは

MTA-STSとは、メールの配送経路上のメールサーバーとメールサーバーの間の暗号化の仕組みを少し強くするためのものです。

具体的には、受信側が、送信サーバーに対して

- STARTTLSを必ず使う
- TLS1.2以上を必ず使う
- 証明書が有効でなければ配送しない

ようにしてもらうことを、お願いする仕組みです。

STARTTLSだけでは、何が不足なのかということについては、後ほど説明します。

例として、sender.example.comからreceiver.example.jpへのメールの配送を考えてみます。

まず、あらかじめ、receiver.example.jpのメール管理者は、自分のところのreceiver.example.jpにメールを送るときには、ちゃんと暗号化してね、とMTA-STSのポリシーでアピールしておきます。
receiver.example.jpのMTA-STSのポリシーを見たsender.example.comのメールサーバーには、receiver.example.jpには暗号化して送る必要があることがわかるので、きちんと暗号化して送ります。また、暗号化できない場合には送りません。

## 設定方法

MTA-STSでは、DNSでMTA-STSを利用することを宣言し、Webでポリシーを広告します。

### DNSの設定

メールのドメインがexample.jpの場合、ホスト名に_mta-stsを追加して、以下のようなTXTレコードを記述します。

```DNS
_mta-sts.example.jp.    IN    TXT    "v=STSv1; id=20201122010000"
```

<dl>
  <dt>v</dt>
  <dd>v=STSv1 はお決まりです。</dd>

  <dt>id</dt>
  <dd>idは1～32文字の英数字です。送信メールサーバーは前回とidが変わっていればポリシーを確認して更新します。値に古い新しいの順序はありません。</dd>
</dl>

### ポリシーの設定

ポリシーはDNSではなくWebに書きます。
httpsでドメインやURLも決まっているのでメール管理者にとっては少し敷居が高いかも知れません。

example.jpに対してMTA-STSのポリシーを公開する場合、次のようなURLになります。

```URL
https://mts-sts.example.jp/.well-known/mta-sts.txt
```

ポリシーはGETで取得したときtext/plainとして取得できる必要があります。

#### ポリシー

メール受信サーバーがmx1.example.jp、mx2.example.jpの2つである場合、次のような記述になります。
改行コードはCR/LFです。

```:mta-sts.txt
version: STSv1
mode: enforce
max_age: 86400
mx: mx1.example.jp
mx: mx2.example.jp
```

mxはワイルドカードが使用できますので、次のような書き方でも構いません。

```:mta-sts.txt
version: STSv1
mode: enforce
max_age: 86400
mx: *.example.jp
```

<dl>
  <dt>version</dt>
  <dd>STSv1 はお決まりです。</dd>

  <dt>mode</dt>
  <dd>STARTTLSに失敗したときの送信サーバーにの動作を記述します。enforce, testing, noneの3種類があります。
  <dl>
    <dt>enforce</dt>
    <dd>STARTTLSが利用できない場合、メールを送信しないようにしてもらいます。
    <dt>testing</dt>
    <dd>STARTTLSが利用できなくても、メールは通常通り送信し、レポートを送信するようにしてもらいます。
    <dt>none</dt>
    <dd>MTA-STSを利用していないように動作してもらいます。MTA-STSを利用したあと、やっぱりやめる場合やオプトアウトで外す場合に使用します。</dd>
  </dd>

  <dt>max_age</dt>
  <dd>送信サーバーがポリシーをキャッシュする期間を設定します。単位は秒で最大31557600(約1年)です。攻撃のリスクを軽減するために可能な限り長くすることが望ましいです。数週間以上が推奨です。</dd>

  <dt>mx</dt>
  <dd>許可するメール受信サーバーのホスト名を記述します。複数指定することができます。*.example.jpのようなワイルドカードドメインも使用できます。</dd>
</dl>

## TLSレポートについて

長くなったので、別に書きます！

## なぜSTARTTLSだけだと不十分なのか

MTA-STSはSTARTTLSをより堅牢にする仕組みです。
では、なぜ、STARTTLSだけでは不十分なのかを解説します。

### Oppotunisticである

STARTTLSはOpportunisticです。
といっても何のことかわからないので、翻訳すると、「STARTTLSは日和見主義です。」
はぁ・・・。やっぱりわかりません。日本語でおkです。

簡単に言うと、STARTTLSは**できればやるけれども、できない場合にはやらない**ということになります。

（日和見というのはできればやりたくない感じがするので、Opportunisticとはちょっと違う気がします。）

どういうことかというと、受信サーバーがSTARTTLSに対応してますよ、と言えば送信サーバーはSTARTTLSを使用して暗号化して送るのですが、受信サーバーがSTARTTLSに対応してると言わなければ、送信サーバーはSTARTTLSを使用せず、平文でメールを送信します。

つまり、受信サーバーがSTARTTLSに対応していれば証明書の検証などが行われるのですが、なりすました偽の受信サーバーはSTARTTLSに対応していると言わないことで、証明書の検証などされることもなく、平文でメールを手に入れられます。

MTA-STSを使えば、STARTTLSの使用が必須となるので、サーバーに平文で送信されることはありません。

### 中間者攻撃に弱い

Opportunisticであるということと同じなのですが、経路上でを盗聴・改ざんされた場合、中間者がSTARTTLSを無効にすることが可能です。(STARTTLS Downgrade Attack)
また、同様に、TLSのハンドシェイクを改ざんすることで、TLS1.1以下にダウングレードさせ、内容を盗聴することも可能なようです。(TLS Protocol Downgrade Attack)

これらのダウングレード攻撃に対して、MTA-STSでは、TLS1.2以上を必須としているので、暗号化のない状態や、脆弱な古い暗号方式で通信されることはありません。

### オレオレ証明書の問題

STARTTLSではRoot CAからのトラストチェーンが切れていたり、有効期限が切れていたり、ホスト名が異なっていても、そのまま通信することがあります。

MTA-STSでは、ポリシーを広告するHTTPSサーバー、受信メールサーバ共に、証明書が有効期限内で、証明書のトラストチェーンがRoot CAまでつながっており、証明書のSANがホスト名と一致しているかどうかを検証しますので、偽の証明書をつかったサーバーに対してメールが送信されることはありません。

## MTA-STSでも守れないこと

DNSのキャッシュポイズニング等でMTA-STSのレコード自体を無効化された場合には、平文で送信されてしまいます。ただし、ポリシーのmax_ageの期間はキャッシュされたポリシーを使用しますので、max_ageを長くしておくのが安全です。

不正な認証局から偽の証明書が発行される場合があります。MTA-STSは証明書のトラストチェーンを信頼していますので、Root CAまでたどれるような偽の証明書の場合、偽物かどうかの区別はできず、悪意のある中間者はメールを詐取可能です。

## おわりに

すでに、メール受信サーバーがTLS1.2に対応しているのであれば、ぜひ、Webサーバーを立ててMTA-STSのポリシーを宣言してみてください。

MTA-STSに対応しているクライアントは当然TLS1.2以上に対応していますので、メールが届くなくなるということはありません。

## 参考資料

[RFC8461](https://tools.ietf.org/html/rfc8461)
[送信ドメイン認証・暗号化 Deep Dive!](https://speakerdeck.com/hirachan/song-xin-domeinren-zheng-an-hao-hua-deep-dive) (自分の発表資料ですが・・・)
