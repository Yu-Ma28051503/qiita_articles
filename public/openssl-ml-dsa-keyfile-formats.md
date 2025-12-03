---
title: OpenSSLにおけるML-DSA鍵ファイルフォーマットの調査
tags:
  - 'security'
  - 'pqc'
  - 'cryptography'
  - 'openssl'
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

本記事は[いちぴろ・エクスプローラのAdvent Calendar](https://qiita.com/advent-calendar/2025/ichipiro-explorer)の2025年12月4日の記事です．

# ML-DSA

ML-DSA(Module Lattice Digital Signature Algorithm)はNISTが標準化したPQC署名アルゴリズムの一つです．
ML-DSAは格子理論に基づく署名アルゴリズムであり，量子コンピュータに対しても耐性があるとされています．
ML-DSAはNISTのPQC標準化プロセスにおいて，最終候補として選ばれたアルゴリズムの一つであり，その安全性と効率性が評価されています．
ML-DSAは特にデジタル署名において，従来の署名アルゴリズムに代わる安全な選択肢として注目されています．

さらに詳しい情報は[FIPS204](https://csrc.nist.gov/pubs/fips/204/final)を参照してください．

# 鍵ファイルフォーマットの調査

## なぜ

wolfSSLでインターンをしているのですが，「ML-DSAって秘密鍵から公開鍵の導出ができるのか?」という疑問が始まりでした(この話は別の記事にする予定です)．
OpenSSLの実装を調査したところ，ML-DSAの鍵ファイルフォーマットがPKCS#8で定義されており，さらに出力フォーマットが複数あることがわかったため，整理することにしました．

## PKCS#8
公開鍵暗号をファイルに出力する際のフォーマットは[PKCS#8(RFC5958)](https://datatracker.ietf.org/doc/html/rfc5958)で定義されています．
構成は次のようになっています:

```text
OneAsymmetricKey ::= SEQUENCE {
  version                   Version,
  privateKeyAlgorithm       PrivateKeyAlgorithmIdentifier,
  privateKey                PrivateKey,
  attributes            [0] Attributes OPTIONAL,
  ...,
  [[2: publicKey        [1] PublicKey OPTIONAL ]],
  ...
}
```

- Version: バージョン情報
- PrivateKeyAlgorithmIdentifier: 鍵アルゴリズムの識別子(OID)
- PrivateKey: 秘密鍵データ
- Attributes: 鍵に関連する属性(オプション)
- PublicKey: 公開鍵データ(オプション)

## OpenSSLドキュメントの確認

[ドキュメント](https://docs.openssl.org/3.5/man7/EVP_PKEY-ML-DSA/#provider-configuration-parameters)を確認してみると，次の6種類の出力フォーマットを選択できる:

| オプション | 説明 |
|---|---|
| `seed-priv` | 32byteのシード値𝜉とFIPS204で示されている秘密鍵のパラメータをPKCS#8で出力(デフォルト) |
| `seed-only` | 32byteのシード値𝜉のみPKCS#8形式で出力 |
| `priv-only` | シード値を含まないFIPS204で示されている秘密鍵のパラメータをPKCS#8で出力 |
| `oqskeypair` | oqsprovider※を使用して公開鍵と秘密鍵が含まれているDERをPKCS#8で出力 |
| `bare-seed` | ASN.1カプセル化なしで32byteのシード値𝜉のみPKCS#8形式で出力 |
| `bare-priv` | ASN.1カプセル化なしで秘密鍵のパラメータをPKCS#8で出力 |

※補足: [OQS(Open Quantum Safe)](https://openquantumsafe.org/)はPQCアルゴリズムの実装を提供しており，プロバイダを使えるようにセットアップする必要がある

ML-DSAの秘密鍵は32byteのシード値𝜉から導出することができます．
そのため，シード値𝜉のみを保存することで秘密鍵パラメータを全て保有しておかなくても鍵の導出が可能となります．


## 鍵生成とASN.1構造体の確認

実際にOpenSSLでML-DSAの秘密鍵を生成してみます．
OpenSSL3.5.0以降のバージョンを用意する必要があります．
おそらく，aptやbrewでインストールされるOpenSSLのバージョンは3.0.x系なので，ソースコードからビルドする必要があります．
[こちらの記事](https://qiita.com/UMA_yuma/items/df5abf49c7a47847a28f)を参考にセットアップしてください．

```bash
openssl version

# OpenSSL 3.5.0 or more recent
```

opensslコマンドのセットアップが完了したら，鍵生成を行ってみます．
出力フォーマットを変更するには`-provparam ml-dsa.output_formats=`オプションを使用します．

```bash
# seed-priv
openssl genpkey -algorithm ml-dsa-44 -provparam ml-dsa.output_formats=seed-priv -out ml-dsa-seed-priv.key -outform PEM

# seed-only
openssl genpkey -algorithm ml-dsa-44 -provparam ml-dsa.output_formats=seed-only -out ml-dsa-seed-only.key -outform PEM

# priv-only
openssl genpkey -algorithm ml-dsa-44 -provparam ml-dsa.output_formats=priv-only -out ml-dsa-priv-only.key -outform PEM

# oqskeypair
openssl genpkey -algorithm ml-dsa-44 -provparam ml-dsa.output_formats=oqskeypair -out ml-dsa-oqskeypair.key -outform PEM -provider oqsprovider

# bare-seed
openssl genpkey -algorithm ml-dsa-44 -provparam ml-dsa.output_formats=bare-seed -out ml-dsa-bare-seed.key -outform PEM

# bare-priv
openssl genpkey -algorithm ml-dsa-44 -provparam ml-dsa.output_formats=bare-priv -out ml-dsa-bare-priv.key -outform PEM
```

生成した鍵ファイルを`openssl asn1parse`コマンドで解析してみます．

```bash
openssl asn1parse -in ml-dsa-seed-priv.key -inform PEM

#  0:d=0  hl=4 l=2622 cons: SEQUENCE
#  4:d=1  hl=2 l=   1 prim: INTEGER           :00
#  7:d=1  hl=2 l=  11 cons: SEQUENCE
#  9:d=2  hl=2 l=   9 prim: OBJECT            :ML-DSA-44
# 20:d=1  hl=4 l=2602 prim: OCTET STRING      [HEX DUMP]:30820A260420482A6726AFAE5F287FD4EF16E5C81AC09E2F7E6...
```

```bash
openssl asn1parse -in ml-dsa-seed-only.key -inform PEM

#  0:d=0  hl=2 l=  52 cons: SEQUENCE
#  2:d=1  hl=2 l=   1 prim: INTEGER           :00
#  5:d=1  hl=2 l=  11 cons: SEQUENCE
#  7:d=2  hl=2 l=   9 prim: OBJECT            :ML-DSA-44
# 18:d=1  hl=2 l=  34 prim: OCTET STRING      [HEX DUMP]:802003A45404C8EC859A07D3BBABCA67880409EAEBA04AA518D8464F279C38E464BA
```

```bash
openssl asn1parse -in ml-dsa-priv-only.key -inform PEM

#  0:d=0  hl=4 l=2584 cons: SEQUENCE
#  4:d=1  hl=2 l=   1 prim: INTEGER           :00
#  7:d=1  hl=2 l=  11 cons: SEQUENCE
#  9:d=2  hl=2 l=   9 prim: OBJECT            :ML-DSA-44
# 20:d=1  hl=4 l=2564 prim: OCTET STRING      [HEX DUMP]:04820A00F7D03415B3724CA96ABA77A0C5B3C73D79C46374254...
```

```bash
openssl asn1parse -in ml-dsa-oqskeypair.key -inform PEM

#  0:d=0  hl=4 l=3896 cons: SEQUENCE
#  4:d=1  hl=2 l=   1 prim: INTEGER           :00
#  7:d=1  hl=2 l=  11 cons: SEQUENCE
#  9:d=2  hl=2 l=   9 prim: OBJECT            :ML-DSA-44
# 20:d=1  hl=4 l=3876 prim: OCTET STRING      [HEX DUMP]:04820F20210E6B075940C604FA70C6F54C904AE5809C45A4422...
```

```bash
openssl asn1parse -in ml-dsa-bare-seed.key -inform PEM

#  0:d=0  hl=2 l=  50 cons: SEQUENCE
#  2:d=1  hl=2 l=   1 prim: INTEGER           :00
#  5:d=1  hl=2 l=  11 cons: SEQUENCE
#  7:d=2  hl=2 l=   9 prim: OBJECT            :ML-DSA-44
# 18:d=1  hl=2 l=  32 prim: OCTET STRING      [HEX DUMP]:5AC56BEF63DF02C52528659C22F6F8A6FF15BE5FC9D41E32A78FAB963980D53D
```

```bash
openssl asn1parse -in ml-dsa-bare-priv.key -inform PEM

#  0:d=0  hl=4 l=2580 cons: SEQUENCE
#  4:d=1  hl=2 l=   1 prim: INTEGER           :00
#  7:d=1  hl=2 l=  11 cons: SEQUENCE
#  9:d=2  hl=2 l=   9 prim: OBJECT            :ML-DSA-44
# 20:d=1  hl=4 l=2560 prim: OCTET STRING      [HEX DUMP]:8AA8DE3B099BDAFF52D8E6AF5F4C66B6E3CDD66BAD0954B072A...
```

## 気付き

上記のASN.1構造体の解析結果から，ML-DSAの秘密鍵ファイルフォーマットはPKCS#8に準拠していることが確認できました．
oqskeypairオプションを使用したときに秘密鍵と公開鍵の両方が出力されるのですが，オプションの公開鍵フィールドではなく秘密鍵フィールドに秘密鍵の一部として出力されていることがわかりました．
`bare-`オプションを使用した場合，ASN.1カプセル化が行われていないため，シード値32byteと秘密鍵2650byteになっており，秘密鍵データが直接出力されていることがわかりました．

# おわりに

本記事では，OpenSSLにおけるML-DSA鍵ファイルフォーマットの調査を行いました．
ML-DSAの秘密鍵ファイルフォーマットはPKCS#8に準拠しており，複数の出力フォーマットが提供されていることがわかりました．
シード値から鍵導出が行えるなら，シード値だけ保存すればいいじゃんと思う一方で，毎回鍵導出を行うコストも無視できないため，ユースケースに応じて適切なフォーマットを選択することが重要です．

この記事を読んで気になったことがあれば，ぜひコメントやフィードバックをお願いします．
