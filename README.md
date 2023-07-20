# TCP_NODELAY serverプログラム

TCP_NODELAYのマニュアル

man 7 tcp

https://man7.org/linux/man-pages/man7/tcp.7.html
のTCP_NODELAYの項目

> TCP_NODELAY
>
> If set, disable the Nagle algorithm.  This means that
> segments are always sent as soon as possible, even if
> there is only a small amount of data.  When not set, data
> is buffered until there is a sufficient amount to send
> out, thereby avoiding the frequent sending of small
> packets, which results in poor utilization of the network.
> This option is overridden by TCP_CORK; however, setting
> this option forces an explicit flush of pending output,
> even if TCP_CORK is currently set.

「できるだけすぐにセグメントが送られる」と書いてある。
write()するたびにwrite()したバイト数だけ送られるという意味ではない。

## serverプログラムの起動方法と動作

```
Usage: server [-p port] [-D] 
-p port: port number (1234)
-D:      use TCP_NODELAY
```

100バイトのデータを連続10回write()する。
10回のwrite()後、sleep(1)で1秒休み。
これを5セット繰り返して終了。

-Dオプションをつけると``TCP_NODELAY``ソケットオプションを
有効にしたソケットに対してwrite()する。

tcpdumpして、適当なクライアントで接続して、
何バイトのパケットが送られるか観察する。

## サーバーを-Dなしで起動したときのtcpdump
-Dなしで起動したサーバー側でtcpdumpしたときの[ログ](no-nodelay.txt)

```
 1	0.000000 0.000000 IP 192.168.10.201.54438 > 192.168.10.200.1234: Flags [S], seq 4249142048, win 29200, options [mss 1460,nop,nop,sackOK], length 0
 2	0.000024 0.000024 IP 192.168.10.200.1234 > 192.168.10.201.54438: Flags [S.], seq 2975989674, ack 4249142049, win 64240, options [mss 1460,nop,nop,sackOK], length 0
 3	0.000149 0.000125 IP 192.168.10.201.54438 > 192.168.10.200.1234: Flags [.], ack 1, win 29200, length 0
 4	0.000684 0.000535 IP 192.168.10.200.1234 > 192.168.10.201.54438: Flags [P.], seq 1:101, ack 1, win 64240, length 100
 5	0.000924 0.000240 IP 192.168.10.201.54438 > 192.168.10.200.1234: Flags [.], ack 101, win 29200, length 0
 6	0.000939 0.000015 IP 192.168.10.200.1234 > 192.168.10.201.54438: Flags [P.], seq 101:1001, ack 1, win 64240, length 900
 7	0.001070 0.000131 IP 192.168.10.201.54438 > 192.168.10.200.1234: Flags [.], ack 1001, win 30600, length 0
 8	1.000921 0.999851 IP 192.168.10.200.1234 > 192.168.10.201.54438: Flags [P.], seq 1001:1101, ack 1, win 64240, length 100
 9	1.001040 0.000119 IP 192.168.10.201.54438 > 192.168.10.200.1234: Flags [.], ack 1101, win 30600, length 0
10	1.001056 0.000016 IP 192.168.10.200.1234 > 192.168.10.201.54438: Flags [P.], seq 1101:2001, ack 1, win 64240, length 900
11	1.001239 0.000183 IP 192.168.10.201.54438 > 192.168.10.200.1234: Flags [.], ack 2001, win 32400, length 0
(以下略)
```

- 1〜3行め: TCP connection
- 4行め: サーバープログラムで100バイト書いたところ。最初なのでパケットがでる。
- 5〜6行め: Nagle アルゴリズムにしたがいackがかえってくるまでデータを送らない。送れる状態になったときには
残り9回のwrite()が終了している。

## サーバーを-D付きで起動してNagleアルゴリズムを無効にした場合
-Dつきで起動したサーバー側でtcpdumpしたときの[ログ](no-nodelay.txt)

```
 1	0.000000 0.000000 IP 192.168.10.201.60920 > 192.168.10.200.1234: Flags [S], seq 1682893850, win 29200, options [mss 1460,nop,nop,sackOK], length 0
 2	0.000032 0.000032 IP 192.168.10.200.1234 > 192.168.10.201.60920: Flags [S.], seq 2445932654, ack 1682893851, win 64240, options [mss 1460,nop,nop,sackOK], length 0
 3	0.000255 0.000223 IP 192.168.10.201.60920 > 192.168.10.200.1234: Flags [.], ack 1, win 29200, length 0
 4	0.000855 0.000600 IP 192.168.10.200.1234 > 192.168.10.201.60920: Flags [P.], seq 1:101, ack 1, win 64240, length 100
 5	0.000869 0.000014 IP 192.168.10.200.1234 > 192.168.10.201.60920: Flags [P.], seq 101:301, ack 1, win 64240, length 200
 6	0.000910 0.000041 IP 192.168.10.200.1234 > 192.168.10.201.60920: Flags [P.], seq 301:1001, ack 1, win 64240, length 700
 7	0.001012 0.000102 IP 192.168.10.201.60920 > 192.168.10.200.1234: Flags [.], ack 101, win 29200, length 0
 8	0.001013 0.000001 IP 192.168.10.201.60920 > 192.168.10.200.1234: Flags [.], ack 301, win 30016, length 0
 9	0.001051 0.000038 IP 192.168.10.201.60920 > 192.168.10.200.1234: Flags [.], ack 1001, win 31500, length 0
10	1.001103 1.000052 IP 192.168.10.200.1234 > 192.168.10.201.60920: Flags [P.], seq 1001:1101, ack 1, win 64240, length 100
11	1.001117 0.000014 IP 192.168.10.200.1234 > 192.168.10.201.60920: Flags [P.], seq 1101:1301, ack 1, win 64240, length 200
12	1.001144 0.000027 IP 192.168.10.200.1234 > 192.168.10.201.60920: Flags [P.], seq 1301:1901, ack 1, win 64240, length 600
13	1.001229 0.000085 IP 192.168.10.200.1234 > 192.168.10.201.60920: Flags [P.], seq 1901:2001, ack 1, win 64240, length 100
14	1.001342 0.000113 IP 192.168.10.201.60920 > 192.168.10.200.1234: Flags [.], ack 1101, win 31500, length 0
15	1.001343 0.000001 IP 192.168.10.201.60920 > 192.168.10.200.1234: Flags [.], ack 1301, win 32900, length 0
16	1.001343 0.000000 IP 192.168.10.201.60920 > 192.168.10.200.1234: Flags [.], ack 2001, win 34300, length 0
(以下略)
```
- 1〜3行め: TCP connection
- 4行め: サーバープログラムで100バイト書いたところ。最初なのでパケットがでる。
- 5〜6行め: Nagle アルゴリズムが無効になっているのでackが返るまえにパケットが送信されている。
送られるバイト数は100バイトwrite()をくりかえしているから100バイトのパケットが送られるかとおもいきや
そうはなっていない。
- 7〜9行め: クライアントプログラムからackが到着
