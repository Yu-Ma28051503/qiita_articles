---
title: IERAE CTF 2024 Write Up
tags:
  - CTF
private: false
updated_at: '2024-09-24T00:33:55+09:00'
id: e1d7e6137a8b2ba751d6
organization_url_name: null
slide: false
ignorePublish: false
---

# 目次
- はじめに
- misc
  - OMG
- rev
  - assignment
- crypto
  - derangment
- おわりに


# はじめに
初CTF writte upです．
今回はIERAE CTF 2024に参加しました．
[IERAE CTF 2024](https://ctftime.org/event/1556)

CTFを始めて1年目のぺーぺーなのでwarmupの3問しか解けなかったのですが，解けた問題をまとめました．
順位はソロ参加で224チーム中113位でした．

# misc
## OMG [warmup]
```問題文
Oh my God!!!My browser history has been hijacked!

オーマイガー！！！ブラウザの履歴が乗っ取られてしまった！

Challenge: http://[host]:[port]
```

Challengeのページにアクセスすると，下のようなページが表示されます．
![OMG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3638216/e80b85d6-8aa8-702c-2c27-1449e6cc46d9.png)

指示に従ってブラウザのバックボタンを33回押すとflagが表示されます．
詳しい原理は分かりませんが，アクセスした際にブラウザの履歴が33回分のページになっているので，バックボタンを押すことでflagが表示されると思われます．

<details><summary>flag</summary>

IERAE{Tr3ndy_4ds.LOL}

</details>

# rev
## assignment [warmup]
```問題文
Assignment is the root of everything in procedural programs.
```

ここでは調査対象のバイナリファイルが渡されるので，静的解析ツール[Ghidra](https://github.com/NationalSecurityAgency/ghidra)を使って解析します．
解析した結果の中からmain関数の中身を確認すると，flagという配列に整数値が格納されていることが分かります．
配列番号順に直して，ASCIIコードに変換するとflagが表示されます．

![解析結果](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3638216/c8acecdd-9c37-b23a-a2c7-9376e6919c2d.png)

<details><summary>flag</summary>

IERAE{s0me_r4nd0m_str1ng_5a9354c}

</details>

# crypto
## derangment [warmup]
```問題文
I've made a secret magic string, perfectly encrypted!

nc [ip address] [port]
```

サーバ側の処理としては，初期化として小文字アルファベットと数字と記号を15文字重複なく選び，ランダムに並べ替えたものを用意し答えとします．セッションを繋ぐたびに答えは変わることに注意です．ユーザ側はヒントを得ることができます．ヒントは答えをシャッフルしたものが出力されます．ここで，シャッフルする際には答えと同じ位置に文字がこないようにしてあります．ヒントを得られる回数は最大で299回です．ヒントを頼りに答えを求めて，合っているとフラグが出力されます．

解法としては，299回のヒントを使って，文字が出現しなかった位置を特定し，その位置に文字を入れることで答えを求めます．これを手入力で行うとセッションが切れてしまうため，pythonスクリプトを使って自動化します．


```python
from pwn import *
import string
import tqdm  # 進捗状況が見れる

# 接続
io = remote('104.199.135.28', 55555)

# ヒントを取得して、hintsに保存
hints = []
for i in tqdm.tqdm(range(299)):
    io.sendlineafter('> ', '1')
    hints.append(io.recvline().decode().strip().split(' ')[1])

# hints[0][i]が何番目に出てくるのかを調べる
index = [[0 for _ in range(15)] for _ in range(15)]
for i in tqdm.tqdm(range(15)):
    c = hints[0][i]
    for hint in hints:
        for j in range(len(hint)):
            if hint[j] == c:
                index[i][j] = 1
                break

ans = ['' for _ in range(15)]
for i in tqdm.tqdm(range(15)):
    for j in range(15):
        if index[i][j] == 0:
            ans[j] += hints[0][i]
            break

result = ''.join(ans)

if result not in hints and len(result) == 15:
    # 2を入力してを答え入力
    io.sendlineafter('> ', '2')
    io.sendlineafter('> ', result)

    # FLAGを取得
    print(io.recvline().decode())
    print(io.recvline().decode())
else:
    print("not found")

# 接続を切断
io.close()

```

<details><summary>flag</summary>

IERAE{th3r35_n0_5uch_th!ng_45_p3rf3ct_3ncrypt!0n}

</details>

# おわりに
今回はIERAE CTF 2024に参加しました．
24時間ぶっ続けでCTFに取り組んだわけではないので十分な成果が出なかったのですが，そもそもの実力不足なので過去問を解いて解いて解きまくろうと思います．
分野としては，暗号の研究をしているのでcryptoの問題を中心に勉強して無双できる日が来るように頑張ります．
あと，IERAE NIGHTにも参加します．
ホワイトハッカーの方々に直接お話を聞けるので楽しみです．
[IERAE NIGHT](https://ierae.connpass.com/event/324203/)
