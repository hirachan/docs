# MTA-STSのススメ (TLSレポート編)

## MTA-STSとは

詳しくは、こちらを参照ください。

[MTA-STSのススメ](https://qiita.com/hirachan/items/d0028da3ebb80b138404)

簡単にいうと、メールの配送経路上のメールサーバーとメールサーバーの間の暗号化の仕組みを少し強くするためのもので、受信側が、送信サーバーに対して

- STARTTLSを必ず使う%
- TLS1.2以上を必ず使う
- 証明書が有効でなければ配送しない

ようにしてもらうことを、お願いする仕組みです。

## TLSレポートとは

MTA-STSが成功した、失敗した、という情報をレポート形式で送ってもらう仕組みです。
MTA-STSのmodeがnone以外の時にレポートが作成されます。

MTA-STSが、と書きましたが、DANEでもレポートを送ってくれるそうなのですが、残念ながら、こちらはまだレポートを手に入れられていません。

## 設定方法

DNSにレポートの送り先を設定します。

### メールで送ってもらう場合

example.jpのレポートをhirano@example.comに送る場合、次のように書きます。

```DNS
_smtp._tls.example.jp.    IN    TXT    "v=TLSRPTv1; rua=mailto:hirano@example.com"
```

レポートの送り先は複数書けますが、DMARCレポートと同じように、`mailto:`をそれぞれに書くのを忘れないようにしてください。

```DNS
_smtp._tls.example.jp.    IN    TXT    "v=TLSRPTv1; rua=mailto:hirano@example.com,mailto:admin@example.com"
```

#### 別ドメインへのレポート送信

DMARCのレポートでは別ドメインへレポートを送信する場合には、レポートを受け取るドメイン側で`example.jp._report._dmarc.example.com`のようなDMARCレコードが必要ですが、MTA-STSでは必要ありません。

### HTTPSで送りつけてもらう場合

example.jpのレポートをhttps://example.jp/reportにアップロードする例です。

```DNS
_smtp._tls.example.jp.    IN    TXT    "v=TLSRPTv1; rua=https://example.jp/report"
```

実は、まだ試したことがないので、どの程度のプロバイダーが対応しているか不明です。

## レポート内容

レポートはメールの場合、JSON形式を圧縮したファイルを添付として送られます。
レポートは基本的にはUTCの00:00～24:00までの24時間単位で、集計と同時ではなく、集計後4時間以内のランダムなタイミングで送信されます。

### 問題がない場合

```json
{
    "organization-name": "Google Inc.",
    "date-range": {
        "start-datetime": "2020-09-07T00:00:00Z",
        "end-datetime": "2020-09-07T23:59:59Z"
    },
    "contact-info": "smtp-tls-reporting@google.com",
    "report-id": "2020-09-07T00:00:00Z_hirano.cc",
    "policies": [
        {
            "policy": {
                "policy-type": "sts",
                "policy-string": [
                    "version: STSv1",
                    "mode: testing",
                    "max_age: 86400",
                    "mx: *.hirano.cc"
                ],
                "policy-domain": "hirano.cc"
            },
            "summary": {
                "total-successful-session-count": 5,
                "total-failure-session-count": 0
            }
        }
    ]
}
```

organization-nameはレポートの作成者、date-rangeはレポートがいつからいつまでのものか、contact-infoはレポートに関する連絡先、report-idはレポート毎の固有のIDです。

policiesはリストになっていますが、これは、MTA-STSとDANEのレポートを同時に表現するためで、MTA-STSだけ設定した場合は、policiesのリストには1つの要素しかありません。

policyには、Webで広告したポリシーがそのまま記載されます。

summaryには成功したセッション数と、失敗したセッション数がそれぞれ書かれます。この場合、5セッション成功して、失敗はなかったということになります。

DMARCレポートとは違ってシンプルでわかりやすい構造になっています。

### 問題がある場合

```json
{
    "organization-name": "Google Inc.",
    "date-range": {
        "start-datetime": "2019-10-01T00:00:00Z",
        "end-datetime": "2019-10-01T23:59:59Z"
    },
    "contact-info": "smtp-tls-reporting@google.com",
    "report-id": "2019-10-01T00:00:00Z_hirano.cc",
    "policies": [
        {
            "policy": {
                "policy-type": "sts",
                "policy-string": [
                    "version: STSv1",
                    "mode: testing",
                    "max_age: 86400",
                    "mx: *.hirano.cc"
                ],
                "policy-domain": "hirano.cc"
            },
            "summary": {
                "total-successful-session-count": 0,
                "total-failure-session-count": 55
            },
            "failure-details": [
                {
                    "result-type": "validation-failure",
                    "sending-mta-ip": "209.85.219.198",
                    "receiving-ip": "210.158.71.76",
                    "receiving-mx-hostname": "ah.hirano.cc",
                    "failed-session-count": 2
                },
                {
                    "result-type": "starttls-not-supported",
                    "sending-mta-ip": "209.85.222.201",
                    "receiving-ip": "210.158.71.76",
                    "receiving-mx-hostname": "ah.hirano.cc",
                    "failed-session-count": 1
                },
                .... 省略 ....
           ]
        }
    ]
}
```

失敗した場合は、失敗の情報がfailure-detailsに記載されます。

たくさんあるので、後ろは省略しましたが、上記の場合、validation-failureが2セッション、starttls-not-supportedが1セッションあったことがわかります。

失敗内容には以下のようなものがあります。

<dl>
  <dt>starttls-not-supported</dt>
  <dd>受信サーバーがSTARTTLSに対応していない</dd>

  <dt>certificate-host-mismatch</dt>
  <dd>証明書のSANの値とホスト名が一致しない</dd>

  <dt>certificate-expired</dt>
  <dd>証明書の有効期限切れ</dd>

  <dt>certificate-not-trusted</dt>
  <dd>認証局が不正、トラストチェーンがRoot CAまでつながらない、など</dd>

  <dt>validation-failure</dt>
  <dd>その他の失敗</dd>
</dl>

## おわりに

MTA-STSを設定したら、TLS-RPTもぜひ同時に設定してください。
MTA-STSのmodeをtestingにして、レポートを読むことで、自分のサーバーの設定ミスがわかったりもします。
自信が付いたら、MTA-STSのmodeをenforceにしましょう。

## 参考資料

[RFC8460 SMTP TLS Reporting](https://tools.ietf.org/html/rfc8460)
[MTA-STSのススメ](https://qiita.com/hirachan/items/d0028da3ebb80b138404)
[送信ドメイン認証・暗号化 Deep Dive!](https://speakerdeck.com/hirachan/song-xin-domeinren-zheng-an-hao-hua-deep-dive) (自分の発表資料ですが・・・)
