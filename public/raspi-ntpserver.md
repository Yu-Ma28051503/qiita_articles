---
title: ラズパイ4にChronyを使ってNTPサーバを実装
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# NTPサーバとは



# インストール

OSのアプデートを行う
```bash
$ sudo apt update | sudo apt upgrade -y
```

Chronyをインストール
```bash
$ sudo apt install chrony
```

設定ファイルの変更
```bash
$ sudo vim /etc/
```

次の箇所を変更
```text
- pool ntp.ubuntu.com        iburst maxsources 4
- pool 0.ubuntu.pool.ntp.org iburst maxsources 1
- pool 1.ubuntu.pool.ntp.org iburst maxsources 1
- pool 2.ubuntu.pool.ntp.org iburst maxsources 2
+ # pool ntp.ubuntu.com        iburst maxsources 4
+ # pool 0.ubuntu.pool.ntp.org iburst maxsources 1
+ # pool 1.ubuntu.pool.ntp.org iburst maxsources 1
+ # pool 2.ubuntu.pool.ntp.org iburst maxsources 2
+ server ntp.nict.jp iburst
+ allow 192.168.100.0/24
```

再起動
```bash
$ systemctl restart chrony
```

動作しているか確認
```bash
$ chronyc sources -v
```

出力が次のようになっていたら完了
```text

  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current best, '+' = combined, '-' = not combined,
| /             'x' = may be in error, '~' = too variable, '?' = unusable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* ntp-b2.nict.go.jp             1   6    17    35  -1223ns[ +597us] +/- 9632us
```

# 参考文献
[https://fabcross.jp/category/make/hobby_raspberrypi/20230620_ntp_server.html]

