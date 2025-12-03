---
title: PQC対応している暗号ライブラリの調査
tags:
  - Security
  - cryptography
  - pqc
private: true
updated_at: '2025-12-03T15:01:10+09:00'
id: 0f8b12e452b361790956
organization_url_name: null
slide: false
ignorePublish: false
---

本記事は[いちぴろ・エクスプローラのAdvent Calendar](https://qiita.com/advent-calendar/2025/ichipiro-explorer)の2025年12月10日の記事です．

# PQC対応している暗号ライブラリの調査

PQC(Post-Quantum Cryptography)の話は盛り上がっているようでなかなか情報が転がっていないものです．
ですが，2030年にはPQC移行を完了を目指すロードマップが発表されており，移行に向けた準備を進める必要があります．
PQC移行を行うために自分たちで暗号ライブラリの実装をするのは大変です．
多くの製品やツールが公開されている，もしくは商用で契約して利用できる暗号ライブラリを利用していることでしょう．

そこで，本記事では公開されており，ソースコードが確認できるPQC対応している暗号ライブラリを調査しました．

# 調査結果

PQCは一言で言っても様々なアルゴリズムの総称に過ぎません．
全てを実装しているライブラリは存在せず，特定のアルゴリズムに対応しているライブラリが多いです．
PQCの中でもNISTがFIPSとして標準化したアルゴリズムのML-KEMとML-DSAに絞って調査しました．
調査漏れは大いにあるので注意してください．

:::alert warning
注意
調査日は2025年11月26日です．
必ず最新バージョンを確認してください．
:::

以下に調査した暗号ライブラリを示します．

| ライブラリ名 | 言語 | ML-KEM | ML-DSA | URL | 特徴・備考 |
|---|---|---|---|---|---|
| OpenSSL | C | 🟢 | 🟢 | https://openssl-library.org/ \ https://github.com/openssl/openssl | デファクトスタンダード |
| liboqs | C/C++ | 🟢 | 🟢 | https://openquantumsafe.org/liboqs/ \ https://github.com/open-quantum-safe/liboqs | PQC実装の先駆者 |
| wolfSSL/wolfCrypt | C | 🟢 | 🟢 | https://github.com/wolfssl/wolfssl | 組み込み向け暗号ライブラリ |
| LibreSSL | C | ❌ | ❌ | https://www.libressl.org/ \ https://github.com/libressl/portable | OpenSSLのフォーク \ OpenBSD標準 |
| SymCrypt | C | 🟢 | 🟢 | https://github.com/microsoft/SymCrypt | Windowsのコア暗号ライブラリ |
| PSA Certified Crypto API  | C | 🟢 | 🟢 | https://arm-software.github.io/psa-api/ \ https://github.com/arm-software/psa-api | 公式のPQC Extensionsを利用することでPQCが使えるようになる \ Mbed TLSが使用 |
| libsodium | C | ❌ | ❌ | https://doc.libsodium.org/ \ https://github.com/jedisct1/libsodium | クロスプラットフォーム対応 |
| Boringssl | C/C++ | 🟢 | 🟢 | https://github.com/google/boringssl | Chrome/Chromium, AndroidのSSLライブラリ |
| Crypto++ | C++ | ❌ | ❌ | https://www.cryptopp.com/ \ https://github.com/weidai11/cryptopp | C++で記述された暗号ライブラリ |
| Botan | C++ | 🟢 | 🟢 | https://github.com/randombit/botan | BSDライセンスで公開されている |
| Conscrypt | Java | ❌ | ▲ | https://source.android.com/docs/core/ota/modular-system/conscrypt?hl=ja \ https://github.com/google/conscrypt/tree/master | 公式に発表はされていないが，ML-DSA実装の動きが見られる |
| Bouncy Castle | Java, C#, Kotlin | 🟢 | 🟢 | https://www.bouncycastle.org/ \ https://github.com/bcgit/bc-csharp | FIPS対応可能 |
| Go標準ライブラリ | Go | 🟢 | ❌ | https://pkg.go.dev/crypto \ https://cs.opensource.google/go/go/+/master:src/crypto/ | ML-DSAが未対応 |
| Cloudflare CIRCL | Go | 🟢 | 🟢 | https://github.com/cloudflare/circl \ https://github.com/cloudflare/circl | PQCとECCを中心に開発 |
| tink-crypto | C++, Java, Go, Python, Objective-C | ❌ | 🟢 | https://developers.google.com/tink?hl=ja \ https://github.com/tink-crypto | KMSとの連携をしやすい |
| Node.js cryptoモジュール | JavaScript | 🟢 | 🟢 | https://nodejs.org/api/crypto.html | Node.js標準の暗号モジュール |
| RustCrypto | Rust | 🟢 | ❌ | https://github.com/RustCrypto | (おそらく)Rust標準 |
| libcrux | Rust | 🟢 | 🟢 | https://cryspen.com/libcrux-library/ \ https://github.com/cryspen/libcrux | 形式検証を受けている |
| pyca | Python | ❌ | ❌ | https://cryptography.io/en/latest/ \ https://github.com/pyca/cryptography | 強気なドメインを使ってる |
| pycryptodome | Python | ❌ | ❌ | https://pypi.org/project/pycryptodome/ \ https://github.com/Legrandin/pycryptodome | CTFで必ず使われる |
| defuse/php-encryption | PHP | ❌ | ❌ | https://docs.flightphp.com/ja/v3/awesome-plugins/php-encryption \ https://github.com/defuse/php-encryption |  |
| CryptX | Perl | ❌ | ❌ | https://metacpan.org/pod/CryptX | |
| gem crypt | Ruby | ❌ | ❌ | http://crypt.finalstep.com.au/ \ https://rubygems.org/gems/crypt/versions/2.2.1?locale=ja | パスワードハッシュの生成のみの機能 |
| Apple CryptoKit | Swift | 🟢 | 🟢 | https://developer.apple.com/documentation/cryptokit/ | Apple公式 |
| Common Lisp crypt | Common Lisp | ❌ | ❌ | https://quickref.common-lisp.net/crypt.html | パスワードハッシュの生成のみの機能 |
| Erlang standard library: crypto | Erlang | ❌ | ❌ | https://security.erlef.org/secure_coding_and_deployment_hardening/crypto | OpenSSL APIを利用 |
| LispKit | [R7RS](https://r7rs.org/) | ❌ | ❌ | https://www.lisppad.app/libraries/lispkit/lispkit-crypto |  |

# おわりに

プログラミング言語に縛られずさまざまな暗号ライブラリの調査を行いました．
暗号を一から実装するのは開発コストが高いため，PQC対応している暗号ライブラリを利用することが望ましいです．
PQC移行を進めるためにも，利用するプログラミング言語がPQC対応している方がスムーズに移行ができるでしょう．
また，PQCは十分に成熟している分野ではないため，今後の動向を十分にチェックする必要があります．

本記事では全ての暗号ライブラリを精査しているわけではないので，実際に利用する際には最新の情報を確認してください．

