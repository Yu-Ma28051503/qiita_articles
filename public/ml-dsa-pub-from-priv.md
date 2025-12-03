---
title: ML-DSAの公開鍵は秘密鍵から導出可能か
tags:
  - 'security'
  - 'cryptography'
  - 'pqc'
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---


# ML-DSAの公開鍵は秘密鍵から導出可能か

## なぜこの話をするのか
このタイトルを見て「自明」「当然だろ」「できないとおかしい」など思うところはたくさんあることでしょう．
情報系の学部では公開鍵暗号の代表例としてRSA暗号を学ぶことが多いはずです．
RSA暗号はかなり単純な構成で，2つの素数$p$,$q$が秘密鍵となり，公開鍵は $n=pq$と$e(= 2^{16}+1)$で構成されます．

しかし，ML-DSAはそう単純ではありません．
ML-DSAの秘密鍵は $sk = (\rho, \mathbf{K}, H(pk, 64), \mathbf{s}_1, \mathbf{s}_2, \mathbf{t}_0)$ で構成されており，公開鍵は $pk = (\rho, \mathbf{t}_1)$ で構成されています．
また，通常の鍵生成アルゴリズムも複雑で，秘密鍵から公開鍵を導出する手順が明示されていません．
そのため，ML-DSAの公開鍵は秘密鍵から導出可能なのかという疑問が浮上しました．

本記事では調査結果を共有します．

## 結論

結論としては，ML-DSAの公開鍵は秘密鍵から導出可能です．

## ML-DSAの鍵生成

ML-DSAの鍵生成アルゴリズムはNISTの[FIPS204](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf)の6.1 ML-DSA Key Generation (Internal)に記載されています．
細かい数学部分や処理は省略して説明します．
鍵生成の流れを示します:

1. 32byteの乱数値(シード値) $\xi$を生成
2. SHAKE256ハッシュ関数を用いて，シード値 $\xi$ からパラメータ $\rho, \rho', K$ を導出
3. $\rho$ から行列 $\mathbf{A}$ を導出
4. $\rho'$ から秘密鍵 $\mathbf{s}_1, \mathbf{s}_2$ を導出
5. $t = \mathbf{A} \cdot \mathbf{s}_1 + \mathbf{s}_2$ を計算
6. $t$ の上位ビットを $\mathbf{t}_1, \mathbf{t}_0$ とする
7. 公開鍵 $pk = (\rho, \mathbf{t}_1)$, 秘密鍵 $sk = (\rho, \mathbf{K}, H(pk, 64), \mathbf{s}_1, \mathbf{s}_2, \mathbf{t}_0)$ とする

以上で，ML-DSAの鍵生成が完了します．

## 導出

ここから秘密鍵の要素から公開鍵を導出する手順を説明します:

1. $\rho$ から行列 $\mathbf{A}$ を導出
2. $t = \mathbf{A} \cdot \mathbf{s}_1 + \mathbf{s}_2$ を計算
3. $t$ の上位ビットを $\mathbf{t}_1$ とする
4. 公開鍵 $pk = (\rho, \mathbf{t}_1)$

鍵生成のプロセスをきちんと追った人にとっては結構シンプルですね．
しかし，RSA暗号のように紙とペンで計算してみようみたいなことができないのが難しいところです．

## OpenSSLの実装確認

OpenSSLではどのように実装しているのか家訓してみましょう．
執筆時点でのコードなので最新情報を必ず確認するようにしてください．

[`openssl/crypto/ml_dsa/ml_dsa_key.c`](https://github.com/openssl/openssl/blob/master/crypto/ml_dsa/ml_dsa_key.c) はML-DSAの鍵に関する処理が実装されています．
その中に `int ossl_ml_dsa_key_public_from_private(ML_DSA_KEY *key)` という関数があります．
さらに，この関数内で `static int public_from_private(const ML_DSA_KEY *key, EVP_MD_CTX *md_ctx, VECTOR *t1, VECTOR *t0)` という関数が呼ばれています．
`public_from_private()` で先ほど説明した導出手順が実装されています．
この関数により，ML-DSAのシード値 $\xi$が鍵ファイルに含まれていなくても，秘密鍵から公開鍵を導出できることが確認できます．

# おわりに

ML-DSAはRSA暗号のように単純ではなく，アルゴリズム全体として複雑です．
多くの暗号学者が関与して設計されているため，安全性は高いと考えられますが，実装や理解が難しい面があります．
今後，システム実装者がPQCを理解するための助けができればと思います．
