---
title: さっそくOpenSSL3.5.0のPQC機能を使ってみる(コマンド編)
tags:
  - Security
  - OpenSSL
  - crypto
  - pqc
private: false
updated_at: '2025-04-11T22:38:22+09:00'
id: df5abf49c7a47847a28f
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
みなさんいつもお世話になっているOpenSSLですが，2025年4月8日にバージョン3.5.0がリリースされました．
NISTのPQC標準化活動によって標準化が決まったML-KEM，ML-DSA，SLH-DSAの3つのアルゴリズムが3.5.0でサポートされました．
PQCの詳細についてはまた後日まとめる予定です．

この記事ではOpenSSLコマンドを使ってML-KEM，ML-DSAを試してみます．
SLH-DSAを試してみようと思ったのですが，コマンドが用意されていなかったので今回は見送りました．

# セットアップ
今回使った環境はUbuntu24.04です．
まずはOpenSSLのプログラムを入手してビルドします．

```bash
sudo apt update
sudo apt install -y gcc make  # 必要なツールをインストール

wget https://www.openssl.org/source/openssl-3.5.0.tar.gz
tar -xzf openssl-3.5.0.tar.gz
cd openssl-3.5.0
./Configure
make
sudo make install

openssl version  # バージョンが出力されると成功
```

最後にバージョンが出力されずエラーが出てしまったら次を試してください．
自分はエラーが出てこれで解決しました．

```bash
LD_LIBRARY_PATH=/usr/local/lib /usr/local/bin/openssl version
LD_LIBRARY_PATH=/usr/local/lib64 /usr/local/bin/openssl version

# 一つ目で正常に出力された場合
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

# 二つ目で正常に出力された場合
export LD_LIBRARY_PATH=/usr/local/lib64:$LD_LIBRARY_PATH

# 最後に反映
export PATH=/usr/local/bin:$PATH
source ~/.bashrc
```

一度ログアウトしてしまうと設定が元に戻ってしまうので注意．

※ 元々インストールされているOpenSSLを完全に置き換えてしまうとバージョンに依存しているプログラムが動かなくなってしまう恐れがあるため，今回は一時的な処置をとっています

# OpenSSL3.5.0のPQC機能を使ってみる

## ML-KEM
ML-KEMは格子暗号をベースにした鍵共有アルゴリズムです．
コマンドライン上で鍵生成，カプセル化，デカプセル化の3つの機能を使ってみます．
まずは秘密鍵と公開鍵を生成します．

**鍵生成**
```bash
openssl genpkey -algorithm ml-kem-512 -out mlkem512.priv -outpubkey mlkem512.pub -outform PEM
```

`mlkem512.priv` と `mlkem512.pub` というファイルが生成されていれば成功です．
次に共有秘密をカプセル化します．

**カプセル化**
```bash
openssl pkeyutl -encap -pubin -inkey mlkem512.pub -keyform pem -secret shared-secret-enc.bin -out mlkem512-ct.bin
```

ここで何も出力されず， `shared-secret-enc.bin` と `mlkem512-ct.bin` というファイルが生成されていれば成功です．
`shared-secret-enc.bin`は共有秘密，`mlkem512-ct.bin`は暗号文です．
最後にカプセル化された暗号文から共有秘密を導出するため，デカプセル化を行います．

**デカプセル化**
```bash
openssl pkeyutl -decap -inkey mlkem512.priv -keyform pem -secret shared-secret-dec.bin -in mlkem512-ct.bin
```

ここで何も出力されず， `shared-secret-dec.bin` というファイルが生成されていれば成功です．
導出された共有秘密が正しく共有されているか確認してみます．
この共有秘密は同じ値である必要があるので `diff`コマンドを使って確認します．

```bash
diff shared-secret-enc.bin shared-secret-dec.bin
```

ここで何も出力されなければ成功です．
不安な人は `xxd`コマンドを使ってバイナリの値を確認してみてください．

```bash
xxd shared-secret-enc.bin
xxd shared-secret-dec.bin
```

`shared-secret-enc.bin`と`shared-secret-dec.bin`の値が同じであれば成功です．
今回はML-KEM512を使用しましたが，ML-KEM768，ML-KEM1024も同様に使用できます．

## ML-DSA
ML-DSAは格子暗号をベースにした署名アルゴリズムです．
鍵生成，署名生成，署名検証の3つの機能を使ってみます．
まずは秘密鍵(署名鍵)と公開鍵(検証鍵)を生成します．

**鍵生成**
```bash
openssl genpkey -algorithm ml-dsa-44 -out mldsa44.priv -outpubkey mldsa44.pub -outform PEM
```

ここで，`mldsa44.priv` と `mldsa44.pub` というファイルが生成されていれば成功です．
次に，署名を生成します．

**署名生成**
```bash
echo "test text" > test.txt  # テスト用のファイルを作成
openssl pkeyutl -sign -in test.txt -inkey mldsa44.priv -keyform pem -out test.sig
```

ここで何も出力されず， `test.sig` というファイルが生成されていれば成功です．
最後に署名 `test.sig` の検証を行います．

**署名検証**
```bash
openssl pkeyutl -verify -pubin -inkey mldsa44.pub -keyform pem -in test.txt -sigfile test.sig
```

ここで，`Signature Verified Successfully`と出力されれば成功です．
今回はML-DSA44を使用しましたが，ML-DSA65，ML-DSA87も同様に使用できます．

## TLSで使ってみる
ML-KEMとML-DSAを使ってTLSで通信してみます．
まずは，ML-DSAを使って　サーバ証明書を生成します．
自己証明書だけでもいいのですが，今回はCA証明書も生成し，サーバ証明書を発行します．
CAはRSA署名を使っています．

```bash
# 前節で生成した鍵はサーバの鍵として使う
mv mldsa44.priv server-key.pem
mv mldsa44.pub server-pubkey.pem

# CAの鍵を生成(RSA)
openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:3072 -out ca-key.pem -outform PEM
openssl req -new -x509 -key ca-key.pem -out ca-cert.pem -days 3650 -subj "/CN=CA" -sha256

# サーバ証明書の生成
openssl req -new -key server-key.pem -out server-csr.pem -subj "/CN=localhost" -sha256
openssl x509 -req -in server-csr.pem -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -days 3650 -sha256
```

ここで，`server-key.pem`がサーバの秘密鍵，`server-cert.pem`がサーバの証明書，`ca-key.pem`がCAの秘密鍵，`ca-cert.pem`がCAの証明書です．
それではTLS通信を行ってみます．
同時に `tcpdump` を使ってパケットを取得し，後でWiresharkで確認します．
ターミナルを3つ立ち上げてください．
1つ目のターミナルでは，サーバを4433番ポートで待ち受けるようにします．

```bash
# サーバ側
openssl s_server -accept 4433 -cert server-cert.pem -key server-key.pem -tls1_3 -keylogfile server-key.log
```

2つ目のターミナルでは，クライアントでTLS接続を行います．
TLSハンドシェイクが完了すると対話モードで入力を待ち受けてしまうので， `printf`コマンドを使ってHTTPリクエストをパイプを使って入力・送信します．

```bash
# クライアント側
printf "GET / HTTP/1.0\r\n\r\n" | \
openssl s_client -connect localhost:4433 -CAfile ca-cert.pem -keylogfile client-key.log
```

3つ目のターミナルでは， `tcpdump`を使ってパケットを取得します．

```bash
# tcpdump
sudo tcpdump -i lo -w pqctls.pcap
```

実行する順番は `tcpdump` → `サーバ` → `クライアント` の順番で実行してください．
実行するといろいろな情報が出てきますが，サーバとクライアントの両方でエラー表示がなければうまくTLS通信ができています．
特に何も表示がないのは，HTMLファイルを設定していないからなので気にしないでください．

## Wiresharkで確認
`tcpdump`で取得したパケットは `pqctls.pcap` というファイルに保存されています．
それでは， `tcpdump`で取得したパケットをWiresharkで確認してみます．

**ML-KEMの確認**
まずはClient Helloの `Extension: supported_groups` のサポートされているグループを確認します．
これは鍵交換を行うための暗号化スキームのグループを示しています．

```text
Supported Group: Unknown (0x11ec)
Supported Group: x25519 (0x001d)
Supported Group: secp256r1 (0x0017)
Supported Group: x448 (0x001e)
Supported Group: secp384r1 (0x0018)
Supported Group: secp521r1 (0x0019)
Supported Group: ffdhe2048 (0x0100)
Supported Group: ffdhe3072 (0x0101)
```

x25519からffdhe3072までは従来通りの楕円曲線暗号です．
ここで，Unknown (0x11ec)というグループが出ていますが，Wiresharkが対応していないためこのような表示になっています．
このグループはML-KEMではなく，X25519とML-KEM512のハイブリッド暗号化スキームです．
PQCはまだ歴史が浅いため，今後どちらかが破られたとしても，もう片方が安全ならば通信が解読されないようにするためにハイブリッド方式を採用していると思われます．
次にServer Helloの `Extension: key_share` > `Extension: Key Share extension` > `Key Share Entry` のサポートされているグループを確認します．

```text
Extension: Key Share Entry: Group: Unknown (4588), Key Exchange length: 1120
  Group: Unknown (4588)
  Key Exchange length: 1120
  Key Exchange [...]: 9ef6...
```

4588は0x11ecと等しいので，Client HelloのUnknown (0x11ec)と同じグループであることがわかります．

**ML-DSAの確認**
通信がうまく行っているのでサーバ証明書の発行が可能になっており，クライアント(OpenSSL3.5.0)でML-DSAを使った証明書の検証ができていることがわかります．
せっかくなので通信を復号してサーバ証明書が本当に検証できるのかを確かめてみます．

TLS1.3ではサーバ証明書は暗号化されているので，Wiresharkで復号する必要があります．
Wiresharkeのメニューから `Edit` → `Preferences` → `Protocols` → `TLS` を選択します．
`Pre-Master Secret log filename` に `server-key.log` を指定します．
`OK`を押して， `pqctls.pcap` を開きます．
すると，TLSのセッションが復号されていることがわかります．
Certificateレコードを探し次の情報まで辿ってください．

```text
`Transport Layer Security`
  -> `TLSv1.3 Recode Layer: Handshake Protocol: Certificate`
    -> `Handshake Protocol: Certificate`
      -> `Certificates`
        -> `Certificate`
```

ここで右クリックをして `Export packet Bytes` を選択，Raw dataで `server-key.der` というファイル名で保存します．
`.bin` が拡張子についてしまうことがあるので `server-key.der` に変更しておきます．
`openssl`コマンドを使って読みやすいように変換して，中身を見てみます．

```bash
# 変換
openssl x509 -inform der -in server-key.der -out server-key.pem

# 中身を確認
openssl x509 -in server-key.pem -text -noout

# 出力
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            69:75:66:f6:b4:14:53:47:1b:80:78:9f:1b:f5:eb:ba:d4:cf:60:2f
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=CA
        Validity
            Not Before: Apr 11 04:29:14 2025 GMT
            Not After : Apr  9 04:29:14 2035 GMT
        Subject: CN=localhost
        Subject Public Key Info:
            Public Key Algorithm: ML-DSA-44
                ML-DSA-44 Public-Key:
                pub:
                    61:d0:4e:bf:dc:7e:56:90:2f:a2:30:e3:5c:97:7b:
                    04:e7:19:27:23:dc:94:9c:fe:1b:02:01:60:75:21:
                    ...(省略)
        X509v3 extensions:
            X509v3 Subject Key Identifier: 
                8F:BF:B0:3A:FB:C9:F8:B3:C6:58:E5:4A:C2:EA:93:1D:BB:CD:C4:C6
            X509v3 Authority Key Identifier: 
                EE:8B:25:7C:58:08:6B:E0:43:0A:A1:78:C0:23:5D:DE:A3:F5:78:4F
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        11:bf:15:84:41:d0:8b:5f:b5:2c:4c:38:b3:cc:d8:e9:3e:aa:
        ca:d6:74:04:2c:65:6a:09:3c:9d:5c:73:2d:01:10:98:ba:b3:
        ...(省略)
```

`Public Key Algorithm` がML-DSA-44となっておりML-DSAを使ったX509証明書であることがわかります．
また， `Signature Algorithm` がsha256WithRSAEncryptionとなっており，CAの署名アルゴリズムがRSAであることがわかります．
このため，現在のCAがRSAやECDSAである場合でも，ML-DSAを使った証明書を発行することができます．

# おわりに
OpenSSL3.5.0のPQC機能を使ってみました．
コマンドライン上で手軽に使えるようになり，これから普及していくと予想されます．
2030年を目標に動いているのでこれからの動向にも注目です．
SLH-DSAはまだコマンドが用意されていないので，実装され次第試してみたいと思います．
また，NISTのPQC標準化活動のround4ではHQCが標準化に選ばれ，2年後に正式な文書を発行する予定らしいです．
liboqsを使ってHQCを試してみるのも面白いかもしれません．
次回はopensslのAPIを使って，C言語でPQCを使ってみる予定です．
