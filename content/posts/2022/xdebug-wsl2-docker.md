---
title: "Docker on WSL2かつエディタがWindowsにある環境でXdebugを動かす"
date: 2022-06-19T17:05:19+09:00
draft: false
---

この記事は[WSL2上のLinuxからホストのWindowsのIPを知るにはmDNSを使うのが楽](/posts/2022/ubuntu-on-wsl2-to-windows-mdns/)の関連記事です。

## 前提条件
一口にWSL2上でDocker使ってるぞ！と言っても、組み合わせがかなり多彩であることに注意する必要があります。
* Docker Desktopを使いWSL2バックエンドが有効で、エディタはWindows上にある
* Docker Desktopを使いWSL2バックエンドが有効で、エディタはWSL2上にある
* Docker Desktopを使わずWSL2上に直接Dockerが起動しており、エディタはWindows上にある
* Docker Desktopを使わずWSL2上に直接Dockerが起動しており、エディタはWSL2上にある

本記事は `WSL2上に直接Dockerが起動しており、エディタはWindows上にある` ユーザーがメインターゲットです。Xdebug自体の構築等については他記事を参照してください。

## `mDNS` 利用編
mDNSを使えば特に設定不要でXdebugが利用できます。

```
xdebug.client_host=${Windowsのホスト名}.local
```

コンテナ再起動で利用できるようになります。接続がうまくいかない場合はファイアウォールでブロックされていないか確認してください。元記事も参考になります。

[WSL2上のLinuxからホストのWindowsのIPを知るにはmDNSを使うのが楽](/posts/2022/ubuntu-on-wsl2-to-windows-mdns/)

これを設定してブレイクポイントに止まればこの記事の残り部分は読む必要ありません。お疲れさまでした。

## `extra-hosts` && `SSH Port Forwarding` 編
何らかの原因がありmDNSを利用できないときは、 `extra-hosts` と `SSH Port Forwarding` を組み合わせることによって目的を達成できます。強いファイアウォールが設定されており例外設定をできない時にはこちらを使用する必要があるかもしれません。

### 実施環境
```shell
> sudo docker version
Server: Docker Engine - Community
 Engine:
  Version:          20.10.17
  API version:      1.41 (minimum version 1.12)
```

### 各種設定
最初に結論を述べてしまうと、コンテナ内からWSL2へのアクセスは `extra-hosts` の設定が、WSL2からWindowsへのアクセスは `SSH port forwarding` の設定がそれぞれ必要になってきます。

```text
+---------------------------------------------------------+
| +------------------------------------+   Host/Windows   |
| | +-----------------+   WSL2/Ubuntu  |                  |
| | |   Docker/PHP    |                | +--------------+ |
| | |                 |                | | IntelliJ     | |
| | |                 |                | |   Port:9099  | |
| | |            -----+-----           | +--------------+ |
| | |            extra-hosts           |                  |
| | |                (1)         ------+------            |
| | |            -----+-----    Port Forwarding           |
| | |                 |               (2)                 |
| | +-----------------+          ------+------            |
| +------------------------------------+                  |
+---------------------------------------------------------+
```

それぞれについて見ていきます。

### `(1) extra-hosts`
まず初めに、Dockerコンテナ内とWSL2上のUbuntuの通信について考えていきます。

すでにDocker Desktop上でXdebugを動かしたことがあるプロジェクトであれば、おそらくxdebugの設定ファイルに `xdebug.client_host=host.docker.internal` と記述されているのではないかと思います。

```
xdebug.client_host=host.docker.internal
```

しかし、このホスト名そのままではLinux上のDocker(=WSL2上のDocker)では正常に名前解決を行うことができません。

```shell
root@3cc27b6f3582:/# ping host.docker.internal
ping: host.docker.internal: Name or service not known
```

そこで、名前解決がされるように `docker-compose.yaml` に `extra_hosts` の追記を行います。

```yaml
  php-fpm:
    image: php:8.0.13-fpm
    # 中略 #
    extra_hosts:
    - "host.docker.internal:host-gateway"
```

> 💡 `extra_hosts` は通常、記述した通りにコンテナ内の `/etc/hosts` を書き換える動作を行いますが、ここに `host-gateway` と指定した場合に限り、ホストにつながるIPアドレスを書き込んでくれます。コードを見る限り、ホスト名が必ず `host.docker.internal` である必要はなさそうですね。
>
> [Support host.docker.internal in dockerd on Linux · moby/moby@92e809a](https://github.com/moby/moby/commit/92e809a6807210a3d1ecd7949314367e82f5b683#diff-f4518e0bed6f8eb7b8a0f874650ddacc308fd0bfce6f0c69878ddf59086c1b28R118)

これで、コンテナ内から `host.docker.internal` にアクセスすることでWSL2上のUbuntuに到達できるようになります。 `/etc/hosts` の値を確認してpingを飛ばしてみます。
```text
root@30f3efe237ed:/usr/share/nginx# cat /etc/hosts
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.1      host.docker.internal <!--これが追加されたアドレス
172.18.0.7      30f3efe237ed

root@30f3efe237ed:/usr/share/nginx# ping host.docker.internal
PING host.docker.internal (172.17.0.1) 56(84) bytes of data.
64 bytes from host.docker.internal (172.17.0.1): icmp_seq=1 ttl=64 time=0.037 ms
64 bytes from host.docker.internal (172.17.0.1): icmp_seq=2 ttl=64 time=0.064 ms
64 bytes from host.docker.internal (172.17.0.1): icmp_seq=3 ttl=64 time=0.064 ms
64 bytes from host.docker.internal (172.17.0.1): icmp_seq=4 ttl=64 time=0.116 ms
64 bytes from host.docker.internal (172.17.0.1): icmp_seq=5 ttl=64 time=0.061 ms
```

Pingが通ればOKです。

#### `(2) Port Forwarding`
`extra-hosts` でコンテナ内とWSL2上のUbuntuとの通信が可能になりました。次はWSL2上のUbuntuからWindowsのエディタとの通信について考えていきます。なお、この作業はWSL上にエディタがある場合は不要です。

まず、セキュリティの観点からデフォルトの設定では外部からのアクセスをSSHを用いて別のホストに転送することができません。仮にポートフォワーディングを行ったとしても、下記のように `localhost` のみがLISTENされます。これでは、Dockerコンテナ内からアクセスがあった際にフォワーディングされることはありません。

```text
COMMAND     PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd      27086 kznrluk    9u  IPv4  66507      0t0  TCP localhost:9002 (LISTEN)
```

これを解決するために、実際にポートフォワーディングを行う前に `/etc/ssh/sshd_config` の `GatewayPorts` の値を `clientspecified` に変更しておきます。

```text
- #GatewayPorts no
+ GatewayPorts clientspecified
```

設定が終わったら `sshd` を起動させます。

```
$ sudo service sshd start
```

次に、Windowsのコンソールに戻ります。コマンドプロンプトでもパワーシェルでもOKです。

実際にWindowsからポートフォワーディングを行います。コマンドの構文とサンプルを以下に記します。

```text
ssh -R ${LISTENするアドレス}:${ポート}:${どのホストに転送するか}:${どのポートに転送するか} ${sshユーザ名}@${sshホスト名}
```

```text
// kznrluK@localhostのすべてのアドレス(0.0.0.0)の9002ポートへの通信をlocalhost:9099に転送する
ssh -R 0.0.0.0:9002:localhost:9099 kznrluk@localhost
```

接続すると通常のSSHと同様にシェルが起動します。SSHが疎通している間、ポートフォワーディングも行われています。
`sudo lsof -i` コマンドで、きちんとポートフォワーディングが行われているか確認できます。

```text
$ sudo lsof -i
COMMAND     PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd      11806 kznrluk    8u  IPv4  27830      0t0  TCP *:9002 (LISTEN)
```

**ポート番号が変換されていることに注意してください！** Ubuntu上のポート9002はWindows上のポート9099に変換されています。Xdebugの設定ファイルで `xdebug.client_port=9099` と指定したり、エディタ上でLISTENするポートを `9002` としないように注意してください。

### トラブルシューティング
`telnet` コマンドを使うことで正常に接続が疎通するかを実際にPHPを動かすことなく確認できます。WSL2上のシェルで確認してみましょう。

ポートフォワーディングができており、かつエディタ側で `9099` ポートをLISTENしている場合は接続できます。 `telnet` で接続できるのにブレイクポイントで止まらない場合は別の問題がありそうです。ポートの設定は合っていますか？ポート変換していることを忘れていませんか？
```text
$ telnet localhost 9002
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
// すぐ切断されなければOK
```

接続できるがすぐ切断されてしまう場合はエディタ側の設定が誤っているかもしれません。ポートはきちんとSSHの接続で指定した値になっていますか？この記事を見て設定したのであれば `9099` をLISTENしているか確認してください。

```text
$ telnet localhost 9002
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Connection closed by foreign host.
```

そもそもポートフォワーディングがうまく言っていない場合は接続できません。SSHの設定から見直してください。
```text
$ telnet localhost 9003
Trying 127.0.0.1...
telnet: Unable to connect to remote host: Connection refused
```

## まとめ
Xdebugはデバッガー側にリクエストが飛ぶような仕組みなのでそこを意識しないと深みにはまってしまいそうです。

`extra-hosts` && `SSH Port Forwarding`編を書いている間に、より良いmDNSを使った方法を知ったため、記事の構成があやふやになってしまいました。記事を書く前に前提を調べておくのは重要ですね。

以上です。