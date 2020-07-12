# OpenVZ7をDRBD9.0 + Pacemakerで冗長化する

OpenVZに関する記事はいくつか見かけるのですが、OpenVZ7になると極端に情報が不足しています。ましてや、冗長化の情報となると、英語で検索しても全く出てきません。

今回は実際に試してみて、うまくいったようにみえる方法をご紹介します。
もっとこうした方がいいとか、ノウハウを隠し持っている方がいらっしゃれば、ぜひご指摘願います。

## 事前準備

### 構成

今回は2台でActive/Standbyの構成を構築します。

1台目(Active): IP: 192.168.111.1
2台目(Standby): IP: 192.168.111.2

Activeな方につながるVIP: 192.168.0.1

のように構成します。

### OpenVZ7のインストール

OpenVZ7は以前のOpenVZとは異なり、RedHatや別のOSでKernelを入れ替えるという方法ではなく、1つのOSとしてインストールします。

https://download.openvz.org/virtuozzo/releases/7.0/x86_64/iso/

まずは、isoをダウンロードしてOSのインストールを行ってください。下記手順をそのまま実行する場合は、古いISOを持っている場合でも、最新をダウンロードするようにしてください。kernelのソースをyumでインストールするときに、古いisoではバージョンが異なり、はまります。

以下の作業は特に断りがない限り、1台目、2台目両方とも同様に実行します。


### 全体アップデート

まずは、気分よく最新にupdateしておきます。

```console
yum -y update
```

### 必要なモジュールのインストール

OpenVZ7用のDRBDモジュールは存在しないので、自力でコンパイルするために必要なモジュールです。

```console
yum -y install gcc gcc-c++ vzkernel-devel flex rpm-build po4a automake autoconf
```

### RPM用ディレクトリの作成

```console
mkdir -p ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
```

### とりあえず、再起動

```console
reboot
```

## DRBDインストール

### DRBD Kernel ModuleのCompile

```console
cd /usr/local/src
wget https://www.linbit.com/downloads/drbd/9.0/drbd-9.0.19-1.tar.gz
tar xvzf drbd-9.0.19-1.tar.gz
cd drbd-9.0.19-1
make
make install
modprobe drbd
cd ..
```

### DRBD コマンドのCompile

```console
wget https://www.linbit.com/downloads/drbd/utils/drbd-utils-9.10.0.tar.gz

tar xvzf drbd-utils-9.10.0.tar.gz
cd drbd-utils-9.10.0

vi drbd.spec.in
```

**34行目あたりに追記**
> %bcond_without sbinsymlinks
> **%undefine with_sbinsymlinks**

```console
./configure
make rpm

cd ..
```

### インストール

```console
cd /root/rpmbuild/RPMS/x86_64
rpm -Uvh drbd-utils-9.10.0-1.vl7.x86_64.rpm drbd-udev-9.10.0-1.vl7.x86_64.rpm drbd-bash-completion-9.10.0-1.vl7.x86_64.rpm
```

## DRBD設定

今回、DRBDで冗長化する領域は、/vzの領域を削除して使用します。
すでにデータが存在しているので、openvzを停止し、データを待避します。

### openvzを停止して、自動で起動しないように

```console
systemctl stop vz prl-disp
systemctl disable vz prl-disp
```

### /vzを待避して消去

```console
cd
cp -a /vz .
umount /vz
dd if=/dev/zero of=/dev/mapper/openvz-vz bs=1M count=1

mv /etc/vz vz/etc-vz
ln -sf /vz/etc-vz /etc/vz
```

## /vzをmountしないように

/etc/fstabで/vzをコメントアウトして起動時にマウントされないようにします。

```sh:/etc/fstab
# /dev/mapper/openvz-vz   /vz  ext4    defaults,noatime,lazytime 1 2

```


### 今回はFirewallは使用しない

```console
systemctl stop firewalld
systemctl disable firewalld
```

### DRBDの設定

```sh:/etc/drbd.d/global_common.conf
    global {
            usage-count no;     <-- noに
            ...
```

/etc/drbd.d/r0.resを新規作成

```:/etc/drbd.d/r0.res

    resource r0 {
        net {
            max-buffers    8000;
            max-epoch-size 8000;
            sndbuf-size 0;
        }
        on server1 {
            device    /dev/drbd0;
            disk      /dev/mapper/openvz-vz;
            address   192.168.111.1:7789;
            meta-disk internal;
        }
        on server2 {
            device    /dev/drbd0;
            disk      /dev/mapper/openvz-vz;
            address   192.168.111.2:7789;
            meta-disk internal;
        }
    }
```

```console
drbdadm create-md r0
drbdadm up r0
```

#### 1台目だけで

```console
drbdadm primary --force r0
```

これで同期が始まる

### 確認

```console
drbdadm status
```

```:1台目
r0 role:Primary
  disk:UpToDate
  server2 role:Secondary
    replication:SyncSource peer-disk:Inconsistent done:0.04
```

```:2台目
r0 role:Secondary
  disk:Inconsistent
  server1 role:Primary
    replication:SyncTarget peer-disk:UpToDate done:0.04
```

完了したら次のようになります。

```:1台目
r0 role:Primary
  disk:UpToDate
  server2 role:Secondary
    peer-disk:UpToDate
```

```:2台目
r0 role:Secondary
  disk:UpToDate
  server1 role:Primary
    peer-disk:UpToDate
```

### 自動起動するようにして再起動

せっかく同期中ですが、ここら辺で一度再起動して、気分をよくしておきます。
同期中に再起動した場合は、最初から同期が始まります。

```console
systemctl enable drbd
reboot
```

### 確認

```console
drbdadm status
```

```:1台目
r0 role:Secondary
  disk:UpToDate
  server2 role:Secondary
    replication:SyncSource peer-disk:Inconsistent done:2.08
```

```:2台目
r0 role:Secondary
  disk:Inconsistent
  server1 role:Secondary
    replication:SyncTarget peer-disk:UpToDate done:2.08
```

両方Secondaryになっていますが、これで正常です。DRBD9.0ではmountした方が自動的にPrimaryになり、unmountしたらSecondaryになります。
ただし、コマンドから手動でPrimaryにした場合はそれ以降自動では切り替わりません。

### ファイルシステムのフォーマット

同期が完了していなくてもファイルシステムのフォーマットでも何でも行って構いません。後でちゃんとつじつまが合います。

```console
mkfs.ext4 /dev/drbd0
```

### 待避したデータを戻す

```console
mount -o noatime,lazytime /dev/drbd0 /vz
cp -a ~/vz/* /vz/
```

ここで```drbdadm status```を実行するとPrimaryになっているはずです。

mountはpacemakerから行うので、umountします。

```console
umount /vz
```

## Pacemakerの設定

### Pacemakerのインストール

```console:両方
yum install -y pacemaker pcs
cp /etc/corosync/corosync.conf.example /etc/corosync/corosync.conf
```

```console:1台目
sed -i -e "s/bindnetaddr: 192.168.1.0/bindnetaddr: 192.168.111.1/" /etc/corosync/corosync.conf
```

```console:2台目
sed -i -e "s/bindnetaddr: 192.168.1.0/bindnetaddr: 192.168.111.2/" /etc/corosync/corosync.conf
```

```console:両方
passwd hacluster

systemctl enable pcsd
systemctl start pcsd
```

```:/etc/hosts(両方)
192.168.111.1 server1
192.168.111.2 server2
```

```console:1台目のみ
pcs cluster auth server1 server2 -u hacluster
Password:

pcs cluster setup --force --name vzcluster server1 server2
pcs cluster start --all
pcs property set stonith-enabled=false
```

### 確認

```console
pcs status
```

しばらくするとOnlineになります。結果はserver2でも同様です。

```
Cluster name: vzcluster
Stack: corosync
Current DC: server1 (version 1.1.19-8.vl7.4-c3c624ea3d) - partition with quorum
Last updated: Tue Sep 17 15:36:49 2019
Last change: Tue Sep 17 15:36:15 2019 by hacluster via crmd on server2

2 nodes configured
0 resources configured

Online: [ server1 server2 ]

No resources


Daemon Status:
  corosync: active/disabled
  pacemaker: active/disabled
  pcsd: active/enabled
```

### リソースの設定

systemdにあるvzとprl-dispを起動しただけでは、なぜか、自動起動設定にしているVPSが起動しないので、仕方なくPacemaker用のリソースファイルを作成します。
prl-dispを起動した後にvzをrestartするとうまく起動するので、prl-dispの依存関係にあるvzが先に起動して、VPS一覧取得に必要な情報が取れないのでVPSが起動せず、prl-disp起動後は一覧情報が取れるのでvzの起動時にVPSが正しく起動するのではと予想しています。
また、/vzをmountする前にprl-dispを起動すると/vzは空なのでmount後もVPSを認識できません。

とりあえず、prl-disp起動後にvzをrestartする方法でうまくいきますが、もっとスマートな解決方法があれば、教えてください。

```shell:/usr/lib/ocf/resource.d/pacemaker/openvz(2台とも)
#!/bin/sh

#######################################################################
# Initialization:

: ${OCF_FUNCTIONS_DIR=${OCF_ROOT}/resource.d/heartbeat}
. ${OCF_FUNCTIONS_DIR}/.ocf-shellfuncs

#######################################################################

MAX_STOP=16
MAX_START=16

VZCTL=/usr/sbin/vzctl
VZQUOTA=/usr/sbin/vzquota
VZLIST=/usr/sbin/vzlist
[ -x ${VZCTL} ] || exit 0

CONFIG_DIR=$OCF_RESKEY_vps/conf

stop_or_start=$__OCF_ACTION

meta_data() {
        cat <<END
<?xml version="1.0"?>
<!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
<resource-agent name="openvz7" version="1.0">
<version>1.0</version>

<parameters>
<parameter name="openvz7" unique="0">
teketou
</parameter>

</parameters>

<actions>
<action name="start"        timeout="1800" />
<action name="stop"         timeout="1800" />
<action name="monitor"      timeout="60" interval="10" depth="0" start-delay="0" />
<action name="meta-data"    timeout="5" />
</actions>
</resource-agent>
END
}

vps_start() {
    set -e
    systemctl start prl-disp
    systemctl restart vz

    exit $OCF_SUCCESS
}

vps_stop() {
    systemctl stop vz prl-disp
    exit $OCF_SUCCESS
}

vps_monitor() {
    if systemctl is-active prl-disp && systemctl is-active vz; then
        exit $OCF_SUCCESS
    else
        exit $OCF_NOT_RUNNING
    fi
}

case $__OCF_ACTION in
  meta-data)      meta_data;;
  start)          vps_start;;
  stop)           vps_stop;;
  monitor)        vps_monitor;;
esac
```

```console
chmod +x /usr/lib/ocf/resource.d/pacemaker/openvz
```

```console:1台目のみ
pcs resource create VIP ocf:heartbeat:IPaddr2 \
    ip=192.168.0.1 cidr_netmask=20 op monitor interval=10s
pcs resource create FS ocf:heartbeat:Filesystem \
    device=/dev/drbd0 directory=/vz fstype=ext4 \
    options=noatime,lazytime \
    op start timeout=60s on-fail=restart \
    op stop timeout=60s on-fail=block \
    op monitor interval=10s timeout=60s on-fail=restart
pcs resource create VZ7 ocf:pacemaker:openvz \
    op monitor interval="30"
pcs resource group add Group-VZ VIP FS VZ7
```

### 確認

```console
pcs status
```

```
Cluster name: vzcluster
Stack: corosync
Current DC: server1 (version 1.1.19-8.vl7.4-c3c624ea3d) - partition with quorum
Last updated: Tue Sep 17 15:51:05 2019
Last change: Tue Sep 17 15:39:47 2019 by root via cibadmin on server1

2 nodes configured
3 resources configured

Online: [ server1 server2 ]

Full list of resources:

 Resource Group: Group-VZ
     VIP        (ocf::heartbeat:IPaddr2):       Started server1
     FS (ocf::heartbeat:Filesystem):    Started server1
     VZ7        (ocf::pacemaker:openvz7):       Started server1

Daemon Status:
  corosync: active/disabled
  pacemaker: active/disabled
  pcsd: active/enabled
```

ResourceがすべてStartedになっていれば、OKです。

これで、準備はすべて完了です。DRBDの同期が終わるのを待って、テストをします。


## テスト

### 準備

Activeな1台目で仮想サーバーを作成します。
作成内容はお好みで。ここでは、単純にコンテナだけ作成します。

```console:1台目
prlctl create test1 --vmtype ct --ostemplate centos-7-x86_64
```

起動して、ファイルでも作っておきます。

```console:1台目(host node)
prlctl start test1
prlctl enter test1
```

```console:test1内
echo testtest > /test.txt
exit
```

2台目で監視しておきます。
ターミナルを2つあげて、DRBDとPacemakerをモニタします。

```console:2台目
watch drbdadm status
```

```console:2台目
watch pcs status
```

### テスト

server1をstandbyモードにします。

```console:どちらでも
pcs cluster standby server1
```

ResourceがStoppedになって、server2で各Resourceが起動します。

```:pcs　statusの結果
Node server1: standby
Online: [ server2 ]

Full list of resources:

 Resource Group: Group-VZ
     VIP        (ocf::heartbeat:IPaddr2):       Started server2
     FS (ocf::heartbeat:Filesystem):    Started server2
     VZ7        (ocf::pacemaker:openvz7):       Started server2
```

test1の中身確認

```console:2台目
server2 # prlctl enter test1
CT-7b049aeb /# cat /test.txt
testtest
```

server1で作成したコンテナがserver2で正しく起動しました。

1台目をONLINEに戻します。

```console:どちらでも
pcs cluster unstandby server1
```

同様に2台目もstandbyにしてみます。

```console:どちらでも
pcs cluster standby server2
```

1台目に移ったか確認します。

```console:1台目
[root@server1 ~]# prlctl list
UUID                                    STATUS       IP_ADDR         T  NAME
{7b049aeb-0bf4-4726-a596-380d8d8ba794}  running      -               CT test1
```

上がってます。いい感じです。

2台目をONLINEに戻しておきましょう。

```console:どちらでも
pcs cluster unstandby server2
```

### その他のテスト

本番稼働前には、サーバ自体をshutdownしたり、いろいろなテストを行ってください。
サーバが再起動した場合には、自動でstandbyにはなりませんので、

```console
pcs cluster start server1
```

のようにして追加してください。

## まとめ
以上、OpenVZ7へのDRBD9.0 + Pacemakerを使った冗長構成の構築方法とその確認方法について説明しました。
うまくいったパターンをご紹介しましたが、おかしなところや、もっとスマートな方法があれば、ぜひご指摘お待ちしています。
