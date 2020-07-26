# Dovecotが「Fatal: mkdir(/usr/local/var/run/dovecot) failed: Read-only file system」と言われて起動できない

## 状況

Dovecot(2.3.10.1)をソースからコンパイルしてインストールして、起動しようとすると、「Fatal: mkdir(/usr/local/var/run/dovecot) failed: Read-only file system」と言われて起動できませんでした。

Permissionの問題かと思い、デレクトリをあらかじめ作ったり、ディレクトリのPermissionを777などにするなどいろいろ試しても、最終的に「master: Fatal: chmod() failed for /usr/local/var/run/dovecot/login: Read-only file system」が出て、にっちもさっちもいかなくなりました。

## 解決方法

/usr/lib/systemd/system/dovecot.service を修正します。

```ini:/usr/lib/systemd/system/dovecot.service
[Service]
...
ProtectSystem=false   <== full を false に変えます
...
```

systemdの設定をリロードします。

```sh
systemctl daemon-reload
```

## 参考

https://forum.iredmail.org/topic13321-iredmail-support-dovecot-error-open-failed-readonly-file-system.html
