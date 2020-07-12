# OpenVZ7 + DRBD + Pacamakerの環境をバージョンアップする

前に、[OpenVZ7をDRBD9.0 + Pacemakerで冗長化する](https://qiita.com/hirachan/items/2aa2beb8fc4832621145)というのを書きましたが、この当時の環境では少し古くなってきてしまいました。

例えば、CentOS8を動かそうとすると、

```console
# prlctl create test --vmtype ct --ostemplate centos-8-x86_64
Creating the Container...
Creating cache
Processing metadata for centos-8-x86_64
Failed to register the CT: PRL_ERR_VZCTL_OPERATION_FAILED (Details: Creating OS template cache for centos-8-x86_64 template
Error: Python directory not found in /vz/pkgenv/rpm414x64
Error: PYTHONPATH variable not defined
Creation of Container private area failed
)
Failed to create the virtual machine.
```

と怒られてしまいます。
そろそろバージョンアップが必要です。

そこで、この環境のOpenVZ7のバージョンを最新にしてみました。

通常OpenVZのバージョンアップは`yum update`をすればいいのですが、この環境はDRBDのカーネルモジュールが入っていますので、こちらも新しいバージョンに合わせる必要があります。

## Secondaryのホストのアップデート

怖いので2台目からやります。

```console
yum update -y
```

## /etc/vzのシンボリックリンクの張り直し

おっと。/vz/etc_vzに向けて張ってあったはずの/etc/vzが実体のディレクトリになってしまいましたので、シンボリックリンクに戻します。

```console
mv /etc/vz /etc/vz.org
ln -sf /vz/etc-vz /etc/vz
```

## DRBDのカーネルモジュールのコンパイル

### 必要なモジュールのインストール

前にやったので入っているはずですが、以下のモジュールが必要です。

```console
yum -y install gcc gcc-c++ vzkernel-devel flex rpm-build po4a automake autoconf
```

### 新しいカーネルバージョンの確認

どこを見るのがベストなのかわからないのですが、/lib/modulesの下を見てみることにします。

```console
# ls /lib/modules
3.10.0-1127.8.2.vz7.151.14
3.10.0-957.12.2.vz7.96.21
```

新しいモジュールは3.10.0-1127.8.2.vz7.151.14のようです。

### DRBDインストール

複数の変更を同時にやると不幸になることが多いので、今回はVZカーネルのバージョンアップに専念することにし、DRBDのバージョンは据え置きます。

```console
cd /usr/local/src
wget https://www.linbit.com/downloads/drbd/9.0/drbd-9.0.19-1.tar.gz
tar xvzf drbd-9.0.19-1.tar.gz
cd drbd-9.0.19-1
export KDIR=/usr/src/kernels/3.10.0-1127.8.2.vz7.151.14
make
make install
depmod 3.10.0-1127.8.2.vz7.151.14
```

### 再起動

```console
shutdown -r now
```

ドキドキしながら見守ります。

### pacemakerの起動

再起動後はPacemakerが自動的に起動しないようにしてあるので、手動で起動します。
PrimaryでもSecondaryでもどちらからでも実行できます。

```console
pcs cluster start server2
```

```
# pcs status

Online: [ server1 server2 ]

Full list of resources:

 Resource Group: Group-VZ
     VIP        (ocf::heartbeat:IPaddr2):       Started server1
     FS (ocf::heartbeat:Filesystem):    Started server1
     VZ7        (ocf::pacemaker:openvz7):       Started server1
```

のようになっていればOKです。

### DRBDの確認

```console:1台目
# drbdadm status
r0 role:Primary
  disk:UpToDate
  server2 role:Secondary
    peer-disk:UpToDate
```

```console:2台目
# drbdadm status
r0 role:Secondary
  disk:UpToDate
  server1 role:Primary
    peer-disk:UpToDate
```

のようになっていればOKです。

## Primaryのホストのアップデート

さて、どうやらうまく行きそうなので、Primaryの方も同様にアップデートします。

### 動作中のVPSをSecondaryに移動

まず、Primaryで動作しているVPSを2台目に移動します。

```console
pcs cluster standby server1
```

次のようになるまで待ちます。

```console
# pcs cluster status

Online: [ server2 ]

Full list of resources:

 Resource Group: Group-VZ
     VIP        (ocf::heartbeat:IPaddr2):       Started server2
     FS (ocf::heartbeat:Filesystem):    Started server2
     VZ7        (ocf::pacemaker:openvz7):       Started server2
```

### OSのバージョンアップとDRBDのカーネルモジュールのインストール

まとめて行きます。

```console
yum update -y

mv /etc/vz /etc/vz.org
ln -sf /vz/etc-vz /etc/vz

yum -y install gcc gcc-c++ vzkernel-devel flex rpm-build po4a automake autoconf
cd /usr/local/src
wget https://www.linbit.com/downloads/drbd/9.0/drbd-9.0.19-1.tar.gz
tar xvzf drbd-9.0.19-1.tar.gz
cd drbd-9.0.19-1
export KDIR=/usr/src/kernels/3.10.0-1127.8.2.vz7.151.14
make
make install
depmod 3.10.0-1127.8.2.vz7.151.14
```

### 再起動

```console
shutdown -r now
```

ドキドキしながら見守ります。

### pacemakerの起動

再起動後はPacemakerが自動的に起動しないようにしてあるので、手動で起動します。
PrimaryでもSecondaryでもどちらからでも実行できます。

```console
pcs cluster start server1
```

```console
# pcs status

Node server1: standby
Online: [ server2 ]

Full list of resources:

 Resource Group: Group-VZ
     VIP        (ocf::heartbeat:IPaddr2):       Started server2
     FS (ocf::heartbeat:Filesystem):    Started server2
     VZ7        (ocf::pacemaker:openvz7):       Started server2
```

のようになっていればOKです。
リソースは、まだ、server2で動作しています。


### リソースを1台目に移動

```console
pcs cluster unstandby server1
```

```console
# pcs status

Online: [ server1 server2 ]

Full list of resources:

 Resource Group: Group-VZ
     VIP        (ocf::heartbeat:IPaddr2):       Started server2
     FS (ocf::heartbeat:Filesystem):    Started server2
     VZ7        (ocf::pacemaker:openvz7):       Started server2
```

```console
pcs cluster standby server2
```

```console
# pcs status

Online: [ server1 server2 ]

Full list of resources:

 Resource Group: Group-VZ
     VIP        (ocf::heartbeat:IPaddr2):       Started server1
     FS (ocf::heartbeat:Filesystem):    Started server1
     VZ7        (ocf::pacemaker:openvz7):       Started server1
```

1台目に寄りました。

### DRBDの確認

念のためDRBDも確認しておきます。

```console:1台目
# drbdadm status
r0 role:Primary
  disk:UpToDate
  server2 role:Secondary
    peer-disk:UpToDate
```

```console:2台目
# drbdadm status
r0 role:Secondary
  disk:UpToDate
  server1 role:Primary
    peer-disk:UpToDate
```

これで完了です。
