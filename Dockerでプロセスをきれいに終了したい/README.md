# Dockerでプロセスをきれいに終了したい

EntrypointがShell Scriptの時、子プログラムを正しく終了させたい。

このようなことを実現する方法です。

前回、[Pythonで終了時に必ず何か実行したい](https://qiita.com/qualitia_cdev/items/f536002791671c6238e3) で、Pythonのプログラムをkillで停止されても、正しくCleanupするようにする方法について書きました。
そのようなきちんとCleanup処理が書かれているプロクラムをDockerで実行したとき、docker stopで正しくClecnupされる方法について見てみます。

## 直接Entrypointで実行する場合

main_process.pyという正しくCleanupされるように書かれたプログラムを直接Entrypointに指定してコンテナを作成してみます。
前回 [Pythonで終了時に必ず何か実行したい](https://qiita.com/qualitia_cdev/items/f536002791671c6238e3)で作成したPythonプログラムを使用します。

```dockerfile:Dockerfile
FROM alpine

RUN apk add python3

WORKDIR /
COPY main_process.py /
RUN chmod +x /main_process.py

ENTRYPOINT /main_process.py
```

```shell
# docker build -t cleanup .
```

実行してみます。
```shell
# docker run -it --rm --name cleanup cleanup
!!!Set up!!!
```

別のターミナルからコンテナを停止します。
```shell
# docker stop cleanup
```

```console
!!!Clean up!!!
```

ちゃんと、Cleanup処理が走りました。
dockerはEntrypointで指定したプログラムをProcess番号1として起動します。
そして、docker stopは、Process番号1に対してkillを実行します。

mail_process.pyはSIGTERMが来るとClean up!!と表示するようになっているので、このように正しく動作しました。


## Shell Script経由で実行する場合

### 単純に実行する

上記のように、直接Entrypointで指定できれば楽なのですが、環境変数から設定ファイルをゴニョゴニョしてからメインのプログラムを実行するという場合もよくあります。
このような場合の動きを確認してみましょう。

main_process.pyを実行するentry.shを作成して、Entrypointにはentry.shを指定します。

```dockerfile:Dockerfile
FROM alpine

RUN apk add python3

WORKDIR /
COPY main_process.py /
COPY entry.sh /
RUN chmod +x /main_process.py
RUN chmod +x /entry.sh

ENTRYPOINT /entry.sh
```

```shell:entry.sh
#!/bin/sh

/main_process.py
```

これをbuildして実行し、さっきと同じようにdocker stopを実行すると、docker stopコマンドが終了しません。
10秒ほどすると、docker stopコマンドが終了し、docker runも同時に終了します。

この時のdocker run側のターミナルは、

```console
# docker run -it --rm --name cleanup cleanup
!!!Set up!!!
#
```

のような状態で、残念ながら、!!!Clean up!!!は表示されません。

10秒はdockerでstopができなかった場合のtimeoutで、dockerはSIGTERMを投げた後、10秒待っても終了しなければ、SIGKILLを投げて強制終了させます。

### execを使用する (解決方法1)

entry.shを以下のように変えて、entry.shのプロセス自身をmain_process.py(正確にはpython3)に置き換えます。

```shell:entry.sh
#!/bin/sh

exec /main_process.py
```

これをbuildして実行し、さっきと同じようにdocker stopを実行すると、今度は、すぐに終了します。

```shell
# docker run -it --rm --name cleanup cleanup
!!!Set up!!!
!!!Clean up!!!
#
```

docker run側のターミナルには、ちゃんとCleanupが表示されています。

execのあと、main_process.pyがプロセス番号1に置き換わっているので、当然といえば当然です。
ただし、終了処理に10秒以上かかるとSIGKILL(kill -9)されてしまうので、Cleanup処理は10秒以内に終わらせることが重要です。

### entry.shでもSIGTERMをトラップする (解決方法2)

```shell:entry.sh
#!/bin/sh

handler() {
    kill ${child_pid}
    wait ${child_pid}
}

trap handler SIGTERM

/main_process.py &
child_pid=$!

wait ${child_pid}
```

子プロセスはバックグラウンドで動作させるようにし、waitで終了を待ちます。
一方で、SIGTERMをトラップし、entry.shがSIGTERMを受け取った場合には、子プロセスをkillし、終了を待つようにします。

先ほどと同様に、実行して、docker stopdしてみると、

```shell
# docker run -it --rm --name cleanup cleanup
!!!Set up!!!
!!!Clean up!!!
```

正しく、Cleanup処理できていることが確認できます。
こちらも、Cleanup処理は10秒以内に終わらせることが重要です。

また、handler()の中身を書き換えれば、子プロセスの終了だけでなく、データをPersistentな領域に移動する、というようなことも可能です。(10秒制限ありますが。)

### 【応用】entry.shから複数の子プロセスの終了を待つようにする

SIGTERMをトラップする方法ではプロセスが1つの場合でも実行をバックグラウンドで行いました。
これを応用すると、複数のプロセスの終了を正しく待つことができそうですので、やってみます。

```shell:entry.sh
#!/bin/sh

handler() {
    kill ${child_pid1}
    kill ${child_pid2}
    wait ${child_pid1}
    wait ${child_pid2}
}

trap handler SIGTERM

/main_process1.py &
child_pid1=$!

/main_process2.py &
child_pid2=$!

wait ${child_pid1}
wait ${child_pid2}
```

2つ並べるだけです。
簡単ですね。

同様にテストしてみます。

```shell
# docker run -it --rm --name cleanup cleanup
!!!Set up!!!
!!!Set up!!!
!!!Clean up!!!
!!!Clean up!!!
#
```

2つのプログラムが正しくCleanup処理を行った後終了したことが確認できました。


##　まとめ

EntrypointにShell Scriptを使う場合、正しく子プロセスを終了させる方法は、

- execでプロセスを置き換える
- SIGTERMをトラップして、子プロセスにSIGTERMを送って待つ

の二通りのやり方があることがわかりました。
