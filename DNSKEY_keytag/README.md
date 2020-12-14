# DNSSECのDNSKEYのKey Tag IDを計算してみる

## そのままでは見えないDNSKEYのKey Tag

DNSSECではDSレコードのKey TagやRRSIGのKey Tagに紐付いたDNSKEYを使って、ごにょごにょします。

digでDSレコードやRRSIGを調べると、レコードの中にKey Tagが入っています。

試しにDSを引いてみます。

```DNS
# dig hiragana.jp ds
...
;; ANSWER SECTION:
hiragana.jp.            6847    IN      DS      1089 13 2 2B6E0A10DB1960663B202339D4A96570BE961136A1D046DE4F060D74 BACB5BB0
```

1089がKey Tagです。

RRSIGの方は、

```console
# dig +dnssec hiragana.jp
...
;; ANSWER SECTION:
hiragana.jp.            282     IN      A       210.158.71.78
hiragana.jp.            282     IN      RRSIG   A 13 2 300 20201223094519 20201209074519 49277 hiragana.jp. t1V+rkruQzOJMOZATP8huwv1GZO+6SOrW+NdpdKkMEFg423NWMOGKLRZ e2ObNFGay1m3oTE+PY92c/YKU7bGtg==
```

49277がKey Tagです。

で、肝心のDNSKEYですが、

```console
# dig hiragana.jp DNSKEY
...
;; ANSWER SECTION:
hiragana.jp.            6786    IN      DNSKEY  257 3 13 pD/DL7KdpYE5+0VF96b+9AL39yXsOJzIWLs5PUP9bIQj9vkdgRKaws22 slHs4T0nXLrzswUHCm3+FY/DpVqPHQ==
hiragana.jp.            6786    IN      DNSKEY  256 3 13 oxiUO5n8Vahk4SqDBslkd0zK8eIFJ6oZCCTmKM+3r49yMGufAjUO8Wb0 vf0tWovHJCEarH60Dvlxw60bE79zLQ==
```

KSKとZSKがありますが、このままではKey Tagがわかりません。

## digでDNSKEYのKey Tagを見る方法

実は、+multilineオプションを付けると、digコマンドでDNSKEYのKey Tagを見ることができます。

```
# dig +multiline hiragana.jp dnskey
...
;; ANSWER SECTION:
hiragana.jp.            6583 IN DNSKEY 256 3 13 (
                                oxiUO5n8Vahk4SqDBslkd0zK8eIFJ6oZCCTmKM+3r49y
                                MGufAjUO8Wb0vf0tWovHJCEarH60Dvlxw60bE79zLQ==
                                ) ; ZSK; alg = ECDSAP256SHA256 ; key id = 49277
hiragana.jp.            6583 IN DNSKEY 257 3 13 (
                                pD/DL7KdpYE5+0VF96b+9AL39yXsOJzIWLs5PUP9bIQj
                                9vkdgRKaws22slHs4T0nXLrzswUHCm3+FY/DpVqPHQ==
                                ) ; KSK; alg = ECDSAP256SHA256 ; key id = 1089
```

いい感じに複数行になるだけではなく、コメントでKeyの種類やアルゴリズム、それにKey Tag IDを表示してくれます。

・・・ということは、何らかの方法で計算できるんやん！

ということで、調べてみました。

## Key Tag IDの計算方法

まずは、RFC4034あたりを当たってみると、ありました！
付録Bです。

[Appendix B.  Key Tag Calculation](https://tools.ietf.org/html/rfc4034#appendix-B)

丁寧にサンプルコードも載っています。
`The code is written for clarity, not efficiency.`なんて書いてありますが、十分ややこしいですね。

```C
   unsigned int
   keytag (
           unsigned char key[],  /* the RDATA part of the DNSKEY RR */
           unsigned int keysize  /* the RDLENGTH */
          )
   {
           unsigned long ac;     /* assumed to be 32 bits or larger */
           int i;                /* loop index */

           for ( ac = 0, i = 0; i < keysize; ++i )
                   ac += (i & 1) ? key[i] : key[i] << 8;
           ac += (ac >> 16) & 0xFFFF;
           return ac & 0xFFFF;
   }
```
ま、でも、ゆっくり読めばやってることは簡単です。
RDATA partというものを2byteずつ区切ってBig Endianで足していけばよさそうです。
その合計に対して、ちょっと不思議な計算をして、最後下位2バイトだけ取り出すだけですね。

では、さっそく計算できるかどうか検証してみます。
そのまま動かしても、おもしろくないので、Pythonで写経してみます。

と、その前に、RDATA partについて確認しておきます。
ここにありました。

[RFC4034 2.1.  DNSKEY RDATA Wire Format](https://tools.ietf.org/html/rfc4034#section-2.1)

```
                        1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |              Flags            |    Protocol   |   Algorithm   |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   /                                                               /
   /                            Public Key                         /
   /                                                               /
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

先ほどの、

```
hiragana.jp.            6786    IN      DNSKEY  257 3 13 pD/DL7KdpYE5+0VF96b+9AL39yXsOJzIWLs5PUP9bIQj9vkdgRKaws22 slHs4T0nXLrzswUHCm3+FY/DpVqPHQ==
```

でいうと、257がFlags、Protocolが3、Algorithが13で、残りのBase64をdecodeしたのがPublic Keyですね。

では、改めて、Pythonで書いてみましょう。
ちょっとPythonぽく、内包表記を使ってみましょう。bit演算は嫌いなので、かけ算や割り算を使ってみました。

```python:keytag.py
import base64

flag = 257
protocol = 3
algorithm = 13
public_key = "pD/DL7KdpYE5+0VF96b+9AL39yXsOJzIWLs5PUP9bIQj9vkdgRKaws22 slHs4T0nXLrzswUHCm3+FY/DpVqPHQ=="

bkey = base64.b64decode(public_key)

ac = flag + protocol * 0x100 + algorithm
ac += sum(bkey[i] * 0x100 + bkey[i + 1] for i in range(0, len(bkey), 2))
ac = (ac + (ac // 0x10000 % 0x10000)) % 0x10000

print(ac)
```

さて、実行してみます。

```console
# python3 a.py
1089
```

KSKなので、先ほどのDSのKey Tag ID 1089と一致しています。

同じように、ZSKの方も見てみます。

```python
flag = 256
public_key = "oxiUO5n8Vahk4SqDBslkd0zK8eIFJ6oZCCTmKM+3r49yMGufAjUO8Wb0 vf0tWovHJCEarH60Dvlxw60bE79zLQ=="
```
の部分を変えます。

```console
# python3 a.py
49277
```

素晴らしい！

RRSIGの49277と一致しました。

## おわりに

DNSKEYのKey Tag IDがどこに隠されているのか、長年の謎が解けました。

めでたしめでたし。
