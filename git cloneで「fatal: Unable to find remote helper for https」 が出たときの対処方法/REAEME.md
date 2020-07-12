# git cloneで「fatal: Unable to find remote helper for 'https'」 が出たときの対処方法

## はじめに

ググるとlibcurl-develをリンクしろというのが多くひっかかりますが、libcurl-develをリンクしていてもエラーが出るケースがあります。

例えば、最小のgit環境を作ろうと、以下のようにした場合、

```Dockerfile:Dockerfile
FROM centos:7 AS src
RUN yum install -y  git

FROM centos:7
COPY --from=src /usr/bin/git /usr/bin/git

ENTRYPOINT ["/bin/sh"]
```

centos7のgitはhttpsにも対応しているのですが、できあがったコンテナではhttpsが利用できません。

```console
# git clone https://github.com/qualitiaco/action-lambda-build-pack-sample.git
Cloning into 'action-lambda-build-pack-sample'...
warning: templates not found /usr/share/git-core/templates
fatal: Unable to find remote helper for 'https'
```

## 原因

文字通り「remote helper for 'https'」というのがないのが原因です。

これは、git-remote-httpsという名前で、/usr/libexec/git-coreの下にあります。

```console
# ls -l /usr/libexec/git-core/git-remote-https
-rwxr-xr-x    4 root     root       1495272 Dec 10 21:39 /usr/libexec/git-core/git-remote-https
```

## 解決方法

/usr/libexec/git-core/git-remote-httpsをコピーします。

OSによってはgit-remote-httpへのsymlinkになっている場合もあるので、その場合、git-remote-httpもコピーします。

## 確認

```Dockerfile:Dockerfile
FROM centos:7 AS src
RUN yum install -y  git

FROM centos:7
COPY --from=src /usr/bin/git /usr/bin/git
COPY --from=src /usr/libexec/git-core/git-remote-https /usr/libexec/git-core/

ENTRYPOINT ["/bin/sh"]
```

```console
# git clone https://github.com/qualitiaco/action-lambda-build-pack-sample.git
Cloning into 'action-lambda-build-pack-sample'...
warning: templates not found /usr/share/git-core/templates
remote: Enumerating objects: 103, done.
remote: Counting objects: 100% (103/103), done.
remote: Compressing objects: 100% (49/49), done.
remote: Total 103 (delta 30), reused 89 (delta 21), pack-reused 0
Receiving objects: 100% (103/103), 11.33 KiB | 0 bytes/s, done.
Resolving deltas: 100% (30/30), done.
```

動きました。

## alpineでもっと小さく

alpineだと、もっと小さくなりますが、追加でいくつかファイルが必要になります。

```Dockerfile:Dockerfile
FROM alpine AS src
RUN apk add git

FROM alpine
COPY --from=src /usr/bin/git /usr/bin/git
COPY --from=src /usr/lib/libpcre2-8.so.0 \
                /usr/lib/libcurl.so.4 \
                /usr/lib/libnghttp2.so.14 \
                /usr/lib/
COPY --from=src /usr/libexec/git-core/git-remote-https /usr/libexec/git-core/
COPY --from=src /etc/ssl/certs/ /etc/ssl/certs/

ENTRYPOINT ["/bin/sh"]
```

これで動きます。

## おわりに

git cloneで「fatal: Unable to find remote helper for 'https'」のエラーが出る場合についての、libcurl-develを入れて再コンパイルしましょう、では解決いない場合の解決方法について説明しました。
