# Pythonで終了時に必ず何か実行したい

途中でディレクトリを作るけど、終了時には残ってて欲しくない。
データベースに仮のデータを置くけど、終了時には残ってて欲しくない。
プログラムが終了したら正常な場合でも、死んだ場合でも、通知して欲しい。

このようなことを実現する方法です。

今回、やりたいこととして、以下のようなものを目指します。

1. 正常な場合にはもちろん実行して欲しい
2. Exceptionが発生しても実行して欲しい
3. Ctrl-Cで止めても実行して欲しい
4. killで止めても実行して欲しい
5. Cleanup処理中は途中で止まらないで欲しい
6. kill -9やSegmentaion Faultはあきらめる

いくつかの方法を試してみます。
結論を急ぐ方のために、先に結論を書いておきます。

## 先に結論

```python
import sys
import time
import signal


def setup():
    print("!!!Set up!!!")


def cleanup():
    print("!!!Clean up!!!")
    # Cleanup処理いろいろ
    time.sleep(10)
    print("!!!Clean up Done!!!")


def sig_handler(signum, frame) -> None:
    sys.exit(1)


def main():
    setup()
    signal.signal(signal.SIGTERM, sig_handler)
    try:
        # いろいろな処理
        time.sleep(60)

    finally:
        signal.signal(signal.SIGTERM, signal.SIG_IGN)
        signal.signal(signal.SIGINT, signal.SIG_IGN)
        cleanup()
        signal.signal(signal.SIGTERM, signal.SIG_DFL)
        signal.signal(signal.SIGINT, signal.SIG_DFL)


if __name__ == "__main__":
    sys.exit(main())
```

## 解説編

### 準備

`setup()`/`clean()`関数とmainの部分を用意しておきます。

```python
def setup():
    print("!!!Set up!!!")


def cleanup():
    print("!!!Clean up!!!")

def main():
    pass

if __name__ == "__main__":
    sys.exit(main())
```

### Case 1. try - finallyを使う

まず思いつくのはtry - finallyを使う方法です。

```python
def main():
    setup()
    try:
        print("Do some jobs")

    finally:
        cleanup()
```

実行してみましょう。

```console
!!!set up!!!
do some jobs
!!!Clean up!!!
```

Clean upされています。
それでは、途中でエラーが発生した場合はどうでしょうか。

```python
def main():
    setup()
    try:
        print(1 / 0)

    finally:
        cleanup()
```

実行してみます。

```console
!!!set up!!!
!!!Clean up!!!
Traceback (most recent call last):
  File "./try-finally.py", line 26, in <module>
    sys.exit(main())
  File "./try-finally.py", line 19, in main
    print(1 / 0)
ZeroDivisionError: division by zero
```

こちらも、Clean upされました。

では、Ctrl-Cで止めた場合はどうなるでしょうか。

```python
def main():
    setup()
    try:
        time.sleep(60)

    finally:
        cleanup()
```

実行して、途中でCtrl-Cを押して止めてみます。

```console
!!!Set up!!!
^C!!!Clean up!!!
Traceback (most recent call last):
  File "./try-finally.py", line 27, in <module>
    sys.exit(main())
  File "./try-finally.py", line 20, in main
    time.sleep(60)
KeyboardInterrupt
```

どうやらうまくいったようです。

では、killで殺した場合はどうでしょうか。

```console
!!!Set up!!!
Terminated
```

これはいけません。これでは、ゴミファイルが残ってしまいます。

#### 結論

try - finallyでは

1. 正常な場合にはもちろん実行して欲しい ⇨ ○
2. Exceptionが発生しても実行して欲しい ⇨ ○
3. Ctrl-Cで止めても実行して欲しい ⇨ ○
4. killで止めても実行して欲しい ⇨ ×

### Case 2. atexitを使う

Pythonには、終了時に実行してくれるというatexitがあります。
これを試してみましょう。

```python
def main():
    setup()
    atexit.register(cleanup)

    print("Do some jobs")
```

実行してみましょう。

```console
!!!Set up!!!
Do some jobs
!!!Clean up!!!
```

うまくいきました。
次は、エラーを出してみます。

```python
def main():
    setup()
    atexit.register(cleanup)

    print(1 / 0)
```

実行してみます。

```console
!!!Set up!!!
Traceback (most recent call last):
  File "./try-finally.py", line 25, in <module>
    sys.exit(main())
  File "./try-finally.py", line 21, in main
    print(1 / 0)
ZeroDivisionError: division by zero
!!!Clean up!!!
```

こちらも、うまくいきました。
try - finallyの場合と比べるとClean upの処理がより後ろになっています。

それでは、Ctrl-Cで止めた場合はどうでしょうか。

```python
def main():
    setup()
    atexit.register(cleanup)

    time.sleep(60)
```

実行して、途中でCtrl-Cで止めてみます。

```console
!!!Set up!!!
^CTraceback (most recent call last):
  File "./try-finally.py", line 25, in <module>
    sys.exit(main())
  File "./try-finally.py", line 21, in main
    time.sleep(60)
KeyboardInterrupt
!!!Clean up!!!
```

いい感じです。

では、最後にkillで止めてみます。

```console
!!!Set up!!!
Terminated
```

残念。try-finallyと同じようにこちらもダメなようです。

#### 結論

atexitでは

1. 正常な場合にはもちろん実行して欲しい ⇨ ○
2. Exceptionが発生しても実行して欲しい ⇨ ○
3. Ctrl-Cで止めても実行して欲しい ⇨ ○
4. killで止めても実行して欲しい ⇨ ×


### Case 3. signalを使う

killコマンドは、SIGTERMというsignalをプロセスに送ります。
また、Ctrl-CはSIGINTというsignalをプロセスに送ります。
これらのsignalをトラップしてみることにしましょう。

```python
def sig_handler(signum, frame) -> None:
    cleanup()
    sys.exit(1)


def main():
    setup()
    signal.signal(signal.SIGTERM, sig_handler)
    signal.signal(signal.SIGINT, sig_handler)

    print("Do some jobs")
```

実行してみます。

```console
!!!Set up!!!
Do some jobs
```

Clean upは実行されません。
外部からsignalが飛んできていないからですね。
あたりまえです。
Exceptionも同様なので、確認は省略します。

では、Ctrl-Cではどうなるか、確認してみます。

```python
def main():
    setup()
    signal.signal(signal.SIGTERM, sig_handler)
    signal.signal(signal.SIGINT, sig_handler)

    time.sleep(60)
```

実行して、途中でCtrl-Cで止めてみます。

```console
!!!Set up!!!
^C!!!Clean up!!!
```

バッチリです。
KeyboardInterruptのExceptionは消えましたが、通常は問題にならないでしょう。
どうしても必要なら、sig_handlerの中でraise(KeyboardInterrupt())できます。


それでは、気になるkillですが、どうなるでしょうか。

```console
!!!Set up!!!
!!!Clean up!!!
```

素晴らしい！
キチンとClean upされました。

#### 結論

1. 正常な場合にはもちろん実行して欲しい ⇨ ×
2. Exceptionが発生しても実行して欲しい ⇨ ×
3. Ctrl-Cで止めても実行して欲しい ⇨ ○
4. killで止めても実行して欲しい ⇨ ○

ということで、try - finallyまたはatexitと、signalを組み合わせればよさそうです。

### Case 4. try - finally と signalを組み合わせて使う

今回は、スコープが狭いという想定で、try - finallyとsignalを組み合わせてみることにします。

```python
def sig_handler(signum, frame) -> None:
    cleanup()
    sys.exit(1)


def main():
    setup()
    signal.signal(signal.SIGTERM, sig_handler)
    try:
        print("do some jobs")

    finally:
        signal.signal(signal.SIGTERM, signal.SIG_DFL)
        cleanup()
```

こんなふうになりました。
Ctrl-Cはtry - finallyでも動作するので、トラップするのはSIGTERMだけにしてみました。
tryを抜けた後は、cleanupは必要ないはずなので、finallyでSIGTERMをデフォルトに戻しておきます。

では、実行してみます。

```console
!!!Set up!!!
do some jobs
!!!Clean up!!!
```

期待通りです。

次は、エラーを出してみます。

```python
def main():
    setup()
    signal.signal(signal.SIGTERM, sig_handler)
    try:
        print(1 / 0)

    finally:
        signal.signal(signal.SIGTERM, signal.SIG_DFL)
        cleanup()


if __name__ == "__main__":
    sys.exit(main())
```

実行してみます。

```console
!!!Set up!!!
!!!Clean up!!!
Traceback (most recent call last):
  File "./try-finally.py", line 33, in <module>
    sys.exit(main())
  File "./try-finally.py", line 26, in main
    print(1 / 0)
ZeroDivisionError: division by zero
```

Clean upされました。

次は、Ctrl-Cを確認します。

```python
def main():
    setup()
    signal.signal(signal.SIGTERM, sig_handler)
    try:
        time.sleep(60)

    finally:
        signal.signal(signal.SIGTERM, signal.SIG_DFL)
        cleanup()
```

実行して、途中でCtrl-Cを押してみます。

```console
!!!Set up!!!
^CClean up!!
Traceback (most recent call last):
  File "./try-finally.py", line 33, in <module>
    sys.exit(main())
  File "./try-finally.py", line 26, in main
    time.sleep(60)
KeyboardInterrupt
```

こちらも、Clean upされました。

さて、killはどうでしょうか。

```console
!!!Set up!!!
!!!Clean up!!!
!!!Clean up!!!
```

えっ！
2回Clean upされてしまいました。

SIGTERMをトラップしたことで、Pythonが強制終了せずに、finallyのところも実行してくれたようです。
ということで、sig_handlerを次のように変更してみます。

```python
def sig_handler(signum, frame) -> None:
    sys.exit(1)
```

handler内でcleanupを呼ぶのをやめて、終了だけするようにしました。

実行して、killしてみます。

```console
!!!Set up!!!
!!!Clean up!!!
```

ようやく期待通りです。

#### 結論

1. 正常な場合にはもちろん実行して欲しい ⇨ ○
2. Exceptionが発生しても実行して欲しい ⇨ ○
3. Ctrl-Cで止めても実行して欲しい ⇨ ○
4. killで止めても実行して欲しい ⇨ ○

### Case 5. Cleanup処理中は止められないようにする

先ほどのやり方でも十分なのですが、Ctrl-Cを押したあと、すぐに止まらないと、Ctrl-Cを連打してしまうのが人情です。
killも何度も実行するかも知れません。

せっかくCleanup処理が動くようにしたのにそれでは台無しです。
ということで、Cleanup中は止められないようにします。

```python
def main():
    setup()
    signal.signal(signal.SIGTERM, sig_handler)
    try:
        time.sleep(60)

    finally:
        signal.signal(signal.SIGTERM, signal.SIG_IGN)
        signal.signal(signal.SIGINT, signal.SIG_IGN)
        cleanup()
        signal.signal(signal.SIGTERM, signal.SIG_DFL)
        signal.signal(signal.SIGINT, signal.SIG_DFL)
```

cleanup()の前にCtrl-Cとkillのシグナルを無視するようにします。
cleanup()が終わったら、デフォルトに戻しておきます。
これで完璧です。

#### 結論

1. 正常な場合にはもちろん実行して欲しい ⇨ ○
2. Exceptionが発生しても実行して欲しい ⇨ ○
3. Ctrl-Cで止めても実行して欲しい ⇨ ○
4. killで止めても実行して欲しい ⇨ ○
5. 終了処理中は途中で止まらないで欲しい ⇨ ○


## おまけ

### try - finally / atexitの使い分け

今回の結果を見る限りtry-catchとatexitの違いは、スコープとタイミングの違いだけのようです。
setupがプログラムの途中で実行されるなら、そのスコープでtry - finallyを使う。
setupがプログラムの一番最初に実行するものなら、atexitを使う。
setupとか関係なく、最後にはいつでも実行して欲しいのなら、atexitを使う。
ということでしょうか。

あと、withを使う方法もあるので、興味のある方は試してみてください。
