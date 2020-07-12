# CentOS8でunboundをソースからコンパイルして使う

現在自宅内でbindを使って権威DNSサーバーとフルリゾルバを混ぜて使っているのですが、リゾルバ部分を分離してunboundで新しく構築してみたいと思います。
せっかくなので、DNSSECの検証もするように設定します。
CentOS8で作りましたが、CentOS7でもdnfをyumにするくらいでできると思います。

## 下準備

OSに入っているunvoundとunbound-libsを念のため削除します。

```sh
dnf remove -y unbound unbound-libs
```

unboundグルーブとunboundユーザーを作成します。OSにすでにunbound-libsがインストールされていた場合にはグループやユーザーがすでに存在しているかも知れません。

```sh
groupadd unbound
useradd -g unbound -s /sbin/nologin unbound
```

必要なライブラリをインストールします。

```sh
dnf install -y gcc openssl-devel libevent-devel expat-devel systemd-devel
```

## インストールと環境設定

unboundのソースをダウンロードしてmake, make installします。

```sh
wget https://www.nlnetlabs.nl/downloads/unbound/unbound-1.10.1.tar.gz
tar xvzf unbound-1.10.1.tar.gz
cd unbound-1.10.1
./configure --enable-systemd --with-libevent
make
make install
```

共有ライブラリを登録します。

```sh
echo /usr/local/lib > /etc/ld.so.conf.d/unbound.conf
ldconfig
```

サービスに登録します。起動するようにするのは、設定が終わってからにします。

```sh
cd contrib
cp unbound.service unbound.socket /usr/lib/systemd/system/
```

unboundユーザが書き込める領域を作成します。

```sh
mkdir -p /usr/local/etc/unbound/runtime
chown unbound:unbound /usr/local/etc/unbound/runtime
```

後で行方不明にならないようにunbound.confを/etcにシンボリックリンクしておきます。

```sh
ln -s /usr/local/etc/unbound/unbound.conf /etc/unbound.conf
```

## 設定ファイル作成

unbound.confを編集します。

```yaml:/usr/local/etc/unbound/unbound.conf
server:
    verbosity: 1
    num-threads: 10
    interface: 0.0.0.0
    outgoing-num-tcp: 100
    incoming-num-tcp: 100
    edns-buffer-size: 512
    max-udp-size: 512
    do-ip6: no
    access-control: 172.30.10.0/24 allow
    access-control: 127.0.0.1 allow
    log-queries: yes
    log-replies: yes
    log-local-actions: yes
    rrset-roundrobin: yes
    minimal-responses: yes
    auto-trust-anchor-file: "/usr/local/etc/unbound/runtime/root.key"
```

access-controlはlocalhostと自宅内の172.30.10.0/24からのみ接続可能なように設定しています。
udpのパケットが分断されたときに書き換えられると嫌なので、4096ではなく、小さく512にしておきます。
ipv6のない環境なので、do-ip6はnoにしておきます。
auto-trust-anchor-fileを指定すると、DNSSECの検証ができるようになります。
rrset-roundrobinをyesにしないとroundrobinしないそうです。
minimal-responsesをyesにすると、必要にないときにはAUTHORITY SECTIONを返さず静かになります。
num-threadsやtcpの数は適当です。環境によって調整してください。
logもお好みで。

### DNSSECの設定

ルートゾーンのDNSSECのkeyを持って来ます。

```sh
sudo -u unbound /usr/local/sbin/unbound-anchor -a /usr/local/etc/unbound/runtime/root.key
```

1日1回取りに行くように設定します。

```console
# crontab -e -u unbound

30 23 * * * /usr/local/sbin/unbound-anchor -a /usr/local/etc/unbound/runtime/root.key
```

### Firewallの設定

FirewallがDNSを通すように設定します。

```console
firewall-cmd --add-service=dns --zone=public --permanent
systemctl reload firewalld
```

### 起動設定

自動で起動するように設定して、起動します。

```sh
systemctl enable unbound
systemctl start unbound
```

## 確認

別のサーバーから確認してみます。

### DNSSECのないホスト

```console
% dig @172.30.10.1 qiita.com

; <<>> DiG 9.11.13-RedHat-9.11.13-5.el8_2 <<>> @172.30.10.1 qiita.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 33714
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;qiita.com.                     IN      A

;; ANSWER SECTION:
qiita.com.              60       IN      A       54.65.107.105
qiita.com.              60       IN      A       54.92.1.147
qiita.com.              60       IN      A       13.112.43.51
```

いい感じです。

### DNSSECのあるホスト

```console
% dig @172.30.10.1 internet.nl

; <<>> DiG 9.11.13-RedHat-9.11.13-5.el8_2 <<>> @172.30.10.1 internet.nl
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2844
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;internet.nl.                   IN      A

;; ANSWER SECTION:
internet.nl.            60      IN      A       62.204.66.10
```

flagsにadが付いています。

### DNSSECが失敗するホスト

```console
% dig @172.30.10.1 dnssec-failed.mufj.jp

; <<>> DiG 9.11.13-RedHat-9.11.13-5.el8_2 <<>> @172.30.10.1 dnssec-failed.mufj.jp
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 45344
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;dnssec-failed.mufj.jp.         IN      A
```

statusがSERVFAILになりました。

## おまけ

元々、自宅のDNSなので、.hiraというエセTLDがあります。
元のDNSサーバーは権威専用のDNSサーバとして残そうと思うので、.hiraは元のDNSサーバ(172.30.10.201, 172.30.10.202)に転送することにします。

```yaml:/usr/local/etc/unbound/unbound.conf
server:
    private-domain: "hira"
    domain-insecure: "hira"
stub-zone:
    name: "hira"
    stub-addr: 172.30.10.201
    stub-addr: 172.30.10.202
```
