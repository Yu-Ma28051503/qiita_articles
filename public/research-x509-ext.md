---
title: X.509証明書拡張を整理
tags:
  - 'security'
  - 'SSL証明書'
  - 'X509'
  - 'RFC'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

最近AIの発展が目覚ましく置いていかれそうになる毎日ですね。
SSL/TLSで使われる証明書には独自の拡張をいろいろ付け加えることができることは知っているでしょうか。
RFC5280では，X.509証明書の構造と機能を定義しており，証明書拡張の役割や基本的な拡張について説明しています。
今回はRFC5280で定義されている証明書拡張を簡単にまとめてみましたので理解の一助になれば幸いです。

# 証明書拡張
## 役割

証明書拡張は通常の証明書フィールドとは別のフィールドで，証明書に追加の属性をユーザまたは公開鍵に関連付けをすることができます。
また，プライベート拡張を使うことで特定のアプリケーションや組織のニーズに合わせたカスタム属性を定義することもできます。

拡張にはクリティカルと非クリティカルの拡張があり，クリティカルな拡張は証明書を処理する際に必ず理解される必要があり，理解されない場合は証明書が無効とされます。
一方，非クリティカルな拡張は理解されない場合でも証明書が有効とされます。

## 基本拡張

基本拡張のOIDは次で定義されている:

```asn1
id-ce OBJECT IDENTIFIER ::=  { joint-iso-ccitt(2) ds(5) 29 (2.5.29) }
```

### Authority Key Identifier(発行者鍵識別子)

Authority Key Identifierは，証明書の署名鍵に対応する公開鍵の識別をするための拡張です。
署名者が複数の署名鍵を持っている場合に利用されます。
CAによって生成される証明書に含まれている必要がありますが，CAの自己証明書として発行する場合は省略が可能となっています。


準拠するCAは，この拡張を非クリティカルにする必要があります。

OIDは， `2.5.29.35` となっており，ASN.1構文は以下のようになっています:

```
id-ce-authorityKeyIdentifier OBJECT IDENTIFIER ::=  { id-ce 35 (2.5.29.35) }

AuthorityKeyIdentifier ::= SEQUENCE {
	keyIdentifier             [0] KeyIdentifier           OPTIONAL,
	authorityCertIssuer       [1] GeneralNames            OPTIONAL,
	authorityCertSerialNumber [2] CertificateSerialNumber OPTIONAL  }

KeyIdentifier ::= OCTET STRING
```

**keyIdentifierのみ入れる場合**

```asn1
extnID = 2.5.29.35
extnValue = AuthorityKeyIdentifier ::= SEQUENCE {
	keyIdentifier             [0] KeyIdentifier           OPTIONAL,
}
```

```der
30 1F
06 03 55 1D 23
04 18
30 16
80 14 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14
```

DERを分解すると，
- 30 1F: SEQUENCE, 長さ31バイト
- 06 03 55 1D 23: OID(2.5.29.35)
- 04 18: extnValue(OCTET STRING), 長さ24バイト
- 30 16: SEQUENCE, 長さ22バイト
- 80 14: [0] (keyIdentifier)，長さ20バイト
- 01...14: kid

**全て入れる場合**

ASN.1構文は同様です。

```der
30 3B
06 03 55 1D 23
04 34
30 32
80 14 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14
A1 0B
86 0A 68 74 74 70 3A 2F 2F 63 61 2F
82 03 01 02 03
```

DERを分解すると，
- 30 3B: SEQUENCE, 長さ59バイト
- 06 03 55 1D 23: OID(2.5.29.35)
- 04 34: extnValue(OCTET STRING), 長さ52バイト
- 30 32: SEQUENCE, 長さ50バイト
- 80 14: [0] (keyIdentifier)，長さ20バイト
- 01...14: kid，長さ20バイト
- A1 0B: [1] (authorityCertIssuer)，長さ11バイト
- 86 0A: GeneralName(URI)
- 68 74 74 70 3A 2F 2F 63 61 2F: "http://ca/"
- 82 03: [2] (authorityCertSerialNumber)，長さ3バイト
- 01 02 03: 証明書シリアル番号

### Subject Key Identifier(主体者鍵識別子)

Subject Key Identifierは，特定の公開鍵を含む証明書を識別するための拡張です。
この拡張は証明書パスの構築を容易にするために必要になります。
CAが発行する証明書にはこの拡張を含める必要があります。
この拡張の値はAuthority Key IdentifierのKeyIdentifierフィールドと一致する必要があります。
ただし，アプリケーションは一致しているかを確認する必要はありません。

公開鍵から鍵識別子を生成するための方法は，RFC5280では以下の2つの手法が書かれています:
1. 公開鍵の値から160ビットSHA-1ハッシュを生成する方法
2. 0b0100の4ビットタイプフィールドに，公開鍵のSHA-1ハッシュの下位60ビットを使用する方法

エンティティ証明書では，複数の証明書を取得している場合に特定の証明書をすばやく識別するための手段として利用されます。

準拠するCAは，この拡張を非クリティカルにする必要があります。

OIDは， `2.5.29.14` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-subjectKeyIdentifier OBJECT IDENTIFIER ::=  { id-ce 14 (2.5.29.14) }

SubjectKeyIdentifier ::= KeyIdentifier
KeyIdentifier ::= OCTET STRING
```

### Key Usage(鍵使用目的)

Key Usageは，証明書に含まれる鍵の暗号化，署名，証明書署名といった使用目的を定義するための拡張です。
例えば，RSA暗号鍵が入っている場合，この鍵は暗号化に使用されるのか，署名に使用されるのかを定義することができます。
1つの用途に絞ることもできますし，複数の用途を定義することもできます。
CAは，公開鍵証明書またはCRLのデジタル署名を検証するために使用される証明書にこの拡張を含める必要があります。

準拠するCAは，この拡張をクリティカルにする必要があります。

OIDは， `2.5.29.15` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-keyUsage OBJECT IDENTIFIER ::=  { id-ce 15 (2.5.29.15) }

KeyUsage ::= BIT STRING {
	digitalSignature      (0),
	nonRepudiation        (1),
	keyEncipherment       (2),
	dataEncipherment      (3),
	keyAgreement          (4),
	keyCertSign           (5),
	cRLSign               (6),
	encipherOnly          (7),
	decipherOnly          (8) }
```

KeyUsageのビットは以下のように使用されます:
- digitalSignature: 証明書署名(5)とCRL(6)以外の**デジタル署名**に使用されることを示すビット
- nonRepudiation: 証明書署名(5)とCRL(6)以外の**否認を防ぐためのサービス**に使用されることを示すビット(最近はcontentCommitmentと呼ばれることが多い)
- keyEncipherment: **鍵暗号化**に使用されることを示すビット
- dataEncipherment: **データ暗号化**に使用されることを示すビット
- keyAgreement: **鍵合意**に使用されることを示すビット
- keyCertSign: **証明書署名**に使用されることを示すビット
- cRLSign: **CRL署名**に使用されることを示すビット
- encipherOnly: 鍵合意(4)が設定されている場合，**鍵合意のデータ暗号化**に使用されることを示すビット
- decipherOnly: 鍵合意(4)が設定されている場合，**鍵合意のデータ復号化**に使用されることを示すビット

### Certificate Policies(証明書ポリシー)

Certificate Policiesは，1つ以上のポリシー識別子(OID)とオプションのポリシーオプションで構成されており，証明書に関連するポリシーを定義するための拡張です。
同じ証明書ポリシー拡張内で複数のポリシー識別子を含めることもできますが，同じポリシーを複数回使用することはできません。
CAが発行する証明書を含む証明書パスのポリシーのセットを制限したくない場合，特別なanyPolicy識別子{ 2 5 29 32 0 }を使用することができます。

RFC5280では，証明書ポリシー作成者と証明書発行者が使用する，CPSポインタ修飾子とユーザ通知修飾子の2つのポリシーオプションが定義されています:
- CPSポインタ修飾子: ポリシー識別子に関連付けられたCPS(Certification Practice Statement)へのURIを提供するためのオプションの修飾子
- ユーザ通知修飾子: 証明書パス検証の結果をユーザに通知するためのオプション修飾子

また，ユーザ通知修飾子には，noticeRefフィールドとexplicitTextフィールドの2つのフィールドが定義されており，どちらか一方または両方が使用されます:
- noticeRefフィールド: 組織が作成した特定の組織名と識別番号のセットを提供するフィールド
- explicitTextフィールド: 証明書に直接最大200文字のテキストが含まれるフィールド

両方のフィールドが使用されているとき，アプリケーションがnoticeRefフィールドにより提供される通知テキストを見つけることができた場合はそのテキストを表示し，見つけることができなかった場合はexplicitTextフィールドにより提供されるテキストを表示する必要があります。

OIDは， `2.5.29.32` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-certificatePolicies OBJECT IDENTIFIER ::=  { id-ce 32 (2.5.29.32) }

anyPolicy OBJECT IDENTIFIER ::=  { id-ce-certificatePolicies 0 }

CertificatePolicies ::= SEQUENCE SIZE (1..MAX) OF PolicyInformation

PolicyInformation ::= SEQUENCE {
	policyIdentifier   CertPolicyId,
	policyQualifiers  SEQUENCE SIZE (1..MAX) OF PolicyQualifierInfo OPTIONAL }

CertPolicyId ::= OBJECT IDENTIFIER

PolicyQualifierInfo ::= SEQUENCE {
	policyQualifierId  PolicyQualifierId,
	qualifier         ANY DEFINED BY policyQualifierId }

-- policyQualifierIds
id-qt          OBJECT IDENTIFIER ::=  { id-pkix 2 }
id-qt-cps      OBJECT IDENTIFIER ::=  { id-qt 1 }
id-qt-unotice  OBJECT IDENTIFIER ::=  { id-qt 2 }

PolicyQualifierId ::= OBJECT IDENTIFIER ( id-qt-cps | id-qt-unotice )

Qualifier ::= CHOICE {
	cpsUri           CPSuri,
	userNotice       UserNotice }

CPSuri ::= IA5String

UserNotice ::= SEQUENCE {
	noticeRef        NoticeReference OPTIONAL,
	explicitText     DisplayText OPTIONAL }

NoticeReference ::= SEQUENCE {
	organization     DisplayText,
	noticeNumbers    SEQUENCE OF INTEGER }

DisplayText ::= CHOICE {
	ia5String        IA5String      (SIZE (1..200)),
	visibleString    VisibleString  (SIZE (1..200)),
	bmpString        BMPString      (SIZE (1..200)),
	utf8String       UTF8String     (SIZE (1..200)) }
```

### Policy Mappings(ポリシーマッピング)

Policy Mappingsは，CA証明書で使用される拡張で，issuerDomainPolicyとsubjectDomainPolicyの2つのペアフィールドを持ち，証明書パス検証の際に発行者ドメインポリシーを主体者ドメインポリシーにマッピングするための拡張です。
発行元となるCAのPKIポリシーと発行先となるCAのPKIポリシーが異なる場合に，この拡張を使用して両者のポリシーをマッピングすることができます。
例えば，発行元CAのPKIポリシーOIDが `1.2.3.4.5` で，発行先CAのPKIポリシーOIDが `10.20.30.40.50` の場合，次のようにマッピングすることができます:

```asn1
PolicyMappings ::= {
	issuerDomainPolicy 1.2.3.4.5,
	subjectDomainPolicy 10.20.30.40.50
}
```

また，他者との合併や買収などの理由で，発行元CAのPKIポリシーと発行先CAのPKIポリシーが異なる場合に，この拡張を使用して両者のポリシーをマッピングすることもできます。

準拠するCAは，この拡張をクリティカルにする必要があります。

マッピングに使われるフィールドは次のような意味を持ちます:
- issuerDomainPolicy: 発行元CA側のポリシーOID
- subjectDomainPolicy: 発行先CA側のポリシーOID

OIDは，`2.5.29.33` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-policyMappings OBJECT IDENTIFIER ::=  { id-ce 33 (2.5.29.33) }

PolicyMappings ::= SEQUENCE SIZE (1..MAX) OF SEQUENCE {
	issuerDomainPolicy   CertPolicyId,
	subjectDomainPolicy  CertPolicyId }
```

### Subject Alternative Name(主体別名)

Subject Alternative Nameは，ID(識別子)を証明書の主体者に関連付けるための拡張です。
DNS名，メールアドレス，URI，IPアドレスなど，主体者を表す複数の識別子を1つの証明書に含めたい場合に使用されます。
別名は公開鍵に確実に関連付けるため，全ての別名はCAによって検証される必要があります。

主体者名(subject)が空で，この拡張のみで主体者を表現する場合，この拡張はクリティカルである必要があります。
一方，subjectにも名前が入っている場合は非クリティカルとして扱われることが一般的です。

GeneralNameにはさまざまな識別子を入れることができ，TLSサーバ証明書では `dNSName` や `ipAddress` がよく利用されます。

OIDは `2.5.29.17` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-subjectAltName OBJECT IDENTIFIER ::=  { id-ce 17 (2.5.29.17) }

SubjectAltName ::= GeneralNames

GeneralNames ::= SEQUENCE SIZE (1..MAX) OF GeneralName

GeneralName ::= CHOICE {
	otherName                       [0]  AnotherName,
	rfc822Name                      [1]  IA5String,
	dNSName                         [2]  IA5String,
	x400Address                     [3]  ORAddress,
	directoryName                   [4]  Name,
	ediPartyName                    [5]  EDIPartyName,
	uniformResourceIdentifier       [6]  IA5String,
	ipAddress                       [7]  OCTET STRING,
	registeredID                    [8]  OBJECT IDENTIFIER }

OtherName ::= SEQUENCE {
	type-id    OBJECT IDENTIFIER,
	value      [0] EXPLICIT ANY DEFINED BY type-id }

EDIPartyName ::= SEQUENCE {
	nameAssigner  [0] DirectoryString OPTIONAL,
	partyName     [1] DirectoryString }
```

### Issuer Alternative Name(発行者別名)

Issuer Alternative Nameは，インターネット上のIDを発行者に関連付けるための拡張です。

準拠するCAは，この拡張を非クリティカルにする必要があります。

OIDは `2.5.29.18` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-issuerAltName OBJECT IDENTIFIER ::=  { id-ce 18 (2.5.29.18) }

IssuerAltName ::= GeneralNames
```

### Subject Directory Attributes(主体者ディレクトリ属性)

Subject Directory Attributesは，証明書の主体者に関連付けられた属性を定義するための拡張です。
関連付けられる属性には，国籍，出身地，性別などが含まれ，[RFC 3739](https://datatracker.ietf.org/doc/html/rfc3739)で解釈されるべき属性が挙げられています。
個人に関する追加属性をX.509証明書に持たせたい場合に使われることがあります。
ここで定義されているものでOIDがいくつか定義されています:
- id-pda-dateOfBirth ::= { id-pkix 9 1 (1.3.6.1.5.5.7.9.1) }
- id-pda-placeOfBirth ::= { id-pkix 9 2 (1.3.6.1.5.5.7.9.2) }
- id-pda-gender ::= { id-pkix 9 3 (1.3.6.1.5.5.7.9.3) }
- id-pda-countryOfCitizenship ::= { id-pkix 9 4 (1.3.6.1.5.5.7.9.4) }
- id-pda-countryOfResidence ::= { id-pkix 9 5 (1.3.6.1.5.5.7.9.5) }

準拠するCAは，この拡張を非クリティカルにする必要があります。

OIDは `2.5.29.9` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-subjectDirectoryAttributes OBJECT IDENTIFIER ::=  { id-ce 9 (2.5.29.9) }

SubjectDirectoryAttributes ::= SEQUENCE SIZE (1..MAX) OF Attribute
```

### Basic Constraints(基本制約)

Basic Constraintsは，証明書がCA証明書であるかどうか，およびこの証明書がどのくらいの深さまでCA証明書を発行できるかを示す拡張です。
中間CA証明書とエンドエンティティ証明書を区別するための重要な拡張であり，証明書パス検証でも利用されます。

CA証明書ではこの拡張を含める必要があり，特に公開鍵が証明書署名に使われる場合はクリティカルである必要があります。
エンドエンティティ証明書では， `cA` を `FALSE` とするか，この拡張を付けずに運用されます。

`cA` はその証明書がCAとして動作できるかどうかを示し， `pathLenConstraint` はそのCAの下に何段まで中間CAをぶら下げられるかを示します。

OIDは，`2.5.29.19` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-basicConstraints OBJECT IDENTIFIER ::=  { id-ce 19 (2.5.29.19) }

BasicConstraints ::= SEQUENCE {
	cA                      BOOLEAN DEFAULT FALSE,
	pathLenConstraint       INTEGER (0..MAX) OPTIONAL }
```

### Name Constraints(名前制約)

Name Constraintsは，証明書が発行できる証明書の名前を制限するための拡張です。
CAが発行する下位証明書の名前空間を制限したい場合に使用されます。
この拡張は自己証明書を除くCA証明書でのみ使用でき，含まれている場合はクリティカルでなければなりません。

`permittedSubtrees` は許可する名前空間を， `excludedSubtrees` は禁止する名前空間を表します。
例えば，特定ドメイン配下のDNS名だけを許可したい場合などに利用されます。

OIDは，`2.5.29.30` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-nameConstraints OBJECT IDENTIFIER ::=  { id-ce 30 (2.5.29.30) }

NameConstraints ::= SEQUENCE {
	permittedSubtrees       [0] GeneralSubtrees OPTIONAL,
	excludedSubtrees        [1] GeneralSubtrees OPTIONAL }

GeneralSubtrees ::= SEQUENCE SIZE (1..MAX) OF GeneralSubtree
GeneralSubtree ::= SEQUENCE {
	base                    GeneralName,
	minimum                 INTEGER DEFAULT 0,
	maximum                 INTEGER OPTIONAL }

BaseDistance ::= INTEGER (0..MAX)
```

### Policy Constraints(ポリシー制約)

Policy Constraintsは，証明書パス検証の際に特定のポリシーを必要とするか，または特定のポリシーを禁止するための拡張です。
この拡張は自己証明書を除くCA証明書でのみ使用でき，この拡張が含まれている場合はクリティカルでなければなりません。

`requireExplicitPolicy` は，何枚先の証明書から明示的なポリシーが必要になるかを示します。
`inhibitPolicyMapping` は，何枚先の証明書からポリシーマッピングを禁止するかを示します。

OIDは，`2.5.29.36` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-policyConstraints OBJECT IDENTIFIER ::=  { id-ce 36 (2.5.29.36) }

PolicyConstraints ::= SEQUENCE {
	requireExplicitPolicy   [0] SkipCerts OPTIONAL,
	inhibitPolicyMapping    [1] SkipCerts OPTIONAL }

SkipCerts ::= INTEGER (0..MAX)
```

### Extended Key Usage(拡張鍵使用目的)

Extended Key Usageは，Key Usage拡張で定義されている使用目的に加えて，証明書の鍵の使用目的をさらに定義するための拡張です。
例えば，TLSサーバ認証やTLSクライアント認証などの特定の使用目的を定義することができます。
この拡張はエンドエンティティ証明書でのみ使用でき，この拡張が含まれている場合は非クリティカルでなければなりません。

OIDは，`2.5.29.37` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-extKeyUsage OBJECT IDENTIFIER ::=  { id-ce 37 (2.5.29.37) }

ExtKeyUsageSyntax ::= SEQUENCE SIZE (1..MAX) OF KeyPurposeId

KeyPurposeId ::= OBJECT IDENTIFIER
```

この拡張を使うための目的は組織によって定義することができますが，[IANA](https://datatracker.ietf.org/doc/html/rfc7299?utm_source=chatgpt.com#section-3.6)または[ITU-TのX.660](https://www.itu.int/itu-t/recommendations/rec.aspx?rec=X.660&utm_source=chatgpt.com)に従う必要があります。
すでに登録されているかをRFC 7299のセクション3.6で確認し，登録されていない場合はX.660に従って登録する必要があります。
この拡張は，証明書発行者のオプションで非クリティカルにもクリティカルにもすることができます。
サーバ認証，クライアント認証，独自OIDなど複数存在する場合，アプリケーションが必要としている拡張のみ確認できればよく，残りの拡張は無視することができます。
証明書に鍵使用目的拡張(Key Usage)と拡張鍵使用目的(Extended Key Usage)の両方が含まれている場合，別々に処理を行い両方の拡張に定義されている使用目的を満たす必要があります。

主要な使用目的は次のように定義されています:

```asn1
anyExtendedKeyUsage OBJECT IDENTIFIER ::=  { id-ce-extKeyUsage 0 (2.5.29.37.0) }

id-kp OBJECT IDENTIFIER ::=  { id-pkix 3 (1.3.6.1.5.5.7.3) }

id-kp-serverAuth OBJECT IDENTIFIER ::=  { id-kp 1 (1.3.6.1.5.5.7.3.1) }
-- TLSサーバ認証に使用されることを示すOID

id-kp-clientAuth OBJECT IDENTIFIER ::=  { id-kp 2 (1.3.6.1.5.5.7.3.2) }
-- TLSクライアント認証に使用されることを示すOID

id-kp-codeSigning OBJECT IDENTIFIER ::=  { id-kp 3 (1.3.6.1.5.5.7.3.3) }
-- コード署名に使用されることを示すOID

id-kp-emailProtection OBJECT IDENTIFIER ::=  { id-kp 4 (1.3.6.1.5.5.7.3.4) }
-- 電子メール保護に使用されることを示すOID

id-kp-timeStamping OBJECT IDENTIFIER ::=  { id-kp 8 (1.3.6.1.5.5.7.3.8) }
-- タイムスタンプのハッシュに使用されることを示すOID

id-kp-OCSPSigning OBJECT IDENTIFIER ::=  { id-kp 9 (1.3.6.1.5.5.7.3.9) }
-- OCSP署名に使用されることを示すOID
```

### CRL Distribution Points(CRL配布点)

CRL配布点は，CRL(Certificate Revocation List)を取得するための方法を定義するための拡張です。
この拡張はエンドエンティティ証明書またはCA証明書に含まれている場合があり，非クリティカルである必要があります。
CRLについては，RFC5280のセクション5.2で定義されているCRL構造を参照してください。
配布点はdistributionPoint，reasons，cRLIssuerの3つのフィールドを持ちます:
- distributionPoint: CRLを取得するための方法を定義するフィールド
- reasons: CRLに含まれる失効理由を定義するフィールド
- cRLIssuer: CRLを発行するCAの識別子を定義するフィールド

OIDは，`2.5.29.31` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-cRLDistributionPoints OBJECT IDENTIFIER ::=  { id-ce 31 (2.5.29.31) }

CRLDistributionPoints ::= SEQUENCE SIZE (1..MAX) OF DistributionPoint

DistributionPoint ::= SEQUENCE {
	distributionPoint       [0] DistributionPointName OPTIONAL,
	reasons                 [1] ReasonFlags OPTIONAL,
	cRLIssuer               [2] GeneralNames OPTIONAL }

DistributionPointName ::= CHOICE {
	fullName                [0] GeneralNames,
	nameRelativeToCRLIssuer [1] RelativeDistinguishedName }

ReasonFlags ::= BIT STRING {
	unused                  (0),
	keyCompromise           (1),
	cACompromise            (2),
	affiliationChanged      (3),
	superseded              (4),
	cessationOfOperation     (5),
	certificateHold          (6),
	privilegeWithdrawn       (7),
	aACompromise             (8) }
```

### Inhibit anyPolicy(任意ポリシー禁止)

anyPolicy禁止拡張はCAに発行された証明書で使用できます。
この拡張は，証明書パス検証の際にanyPolicy識別子を禁止するための拡張です。
この拡張が含まれている場合はクリティカルでなければなりません。

`SkipCerts` は，何枚先の証明書から `anyPolicy` を無効として扱うかを表します。

OIDは，`2.5.29.54` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-inhibitAnyPolicy OBJECT IDENTIFIER ::=  { id-ce 54 (2.5.29.54) }

InhibitAnyPolicy ::= SkipCerts

SkipCerts ::= INTEGER (0..MAX)
```

### Freshest CRL(a.k.a. Delta CRL Distribution Points)(最新CRL(別名: デルタCRL配布点))

最新CRL拡張はCRLに使用される拡張で，CRL配布点拡張と同様の構造を持ち，最新のCRL(デルタCRL)の取得方法を示すための拡張です。
この拡張が含まれている場合は非クリティカルでなければなりません。

完全なCRLではなく差分CRLを取得したい場合に参照され，構造自体は `CRLDistributionPoints` を再利用しています。

OIDは，`2.5.29.46` となっており，ASN.1構文は以下のようになっています:

```asn1
id-ce-freshestCRL OBJECT IDENTIFIER ::=  { id-ce 46 (2.5.29.46) }

FreshestCRL ::= CRLDistributionPoints
```

## Private Internet Extensions(プライベートインターネット拡張)

RFC5280では2つのプライベートインターネット拡張が定義されています。
PKI用に定義された将来の拡張も `id-pe` OIDの下に定義されることが期待されています。
OIDは次のように定義されています:

```asn1
id-pkix OBJECT IDENTIFIER ::=  { iso(1) identified-organization(3) dod(6) internet(1) security(5) mechanisms(5) pkix(7) (1.3.6.1.5.5.7) }

id-pe OBJECT IDENTIFIER ::=  { id-pkix 1 (1.3.6.1.5.5.7.1) }
```

### Authority Information Access(発行者情報アクセス)

発行者情報アクセスは，証明書の発行者に関連する情報へのアクセス方法を定義するための拡張です。
証明書を発行したCAの証明書またはOCSPレスポンスヘッダへのアクセス方法を定義するために使用されます。
この拡張はエンドエンティティ証明書またはCA証明書に含まれている場合があり，非クリティカルである必要があります。

`accessMethod` には情報の種類を， `accessLocation` にはその取得先を入れます。
代表的な `accessMethod` として，発行者CA証明書の取得先を示す `id-ad-caIssuers` と，OCSPレスポンダを示す `id-ad-ocsp` があります。

OIDは，`1.3.6.1.5.5.7.1.1` となっており，ASN.1構文は以下のようになっています:

```asn1
id-pe-authorityInfoAccess OBJECT IDENTIFIER ::=  { id-pe 1 (1.3.6.1.5.5.7.1.1) }

AuthorityInfoAccessSyntax ::= SEQUENCE SIZE (1..MAX) OF AccessDescription

AccessDescription ::= SEQUENCE {
	accessMethod          OBJECT IDENTIFIER,
	accessLocation        GeneralName }

id-ad OBJECT IDENTIFIER ::=  { id-pkix 48 (1.3.6.1.5.5.7.48) }

id-ad-caIssuers OBJECT IDENTIFIER ::=  { id-ad 2 (1.3.6.1.5.5.7.48.2) }
id-ad-ocsp OBJECT IDENTIFIER ::=  { id-ad 1 (1.3.6.1.5.5.7.48.1) }
```

### Subject Information Access(主体者情報アクセス)

主体者情報アクセス拡張は，証明書の主体者に関連する情報へのアクセス方法を定義するための拡張です。
主体者がCAの場合，証明書検証のための情報とCAポリシーデータが含まれることがあります。
主体者がエンティティの場合，情報が提供されるサービスタイプとアクセス方法が含まれることがあります。
この拡張は非クリティカルとしてマークする必要があります。

Authority Information Accessと同様に， `accessMethod` が情報の種類， `accessLocation` が取得先を表します。
代表例として，CAリポジトリを示す `id-ad-caRepository` やタイムスタンプサービスを示す `id-ad-timeStamping` が使われます。

OIDは，`1.3.6.1.5.5.7.1.11` となっており，ASN.1構文は以下のようになっています:

```asn1
id-pe-subjectInfoAccess OBJECT IDENTIFIER ::=  { id-pe 11 (1.3.6.1.5.5.7.1.11) }

SubjectInfoAccessSyntax ::= SEQUENCE SIZE (1..MAX) OF AccessDescription

AccessDescription ::= SEQUENCE {
	accessMethod          OBJECT IDENTIFIER,
	accessLocation        GeneralName }

id-ad OBJECT IDENTIFIER ::=  { id-pkix 48 (1.3.6.1.5.5.7.48) }

id-ad-caRepository OBJECT IDENTIFIER ::=  { id-ad 5 (1.3.6.1.5.5.7.48.5) }

id-ad-timeStamping OBJECT IDENTIFIER ::=  { id-ad 3 (1.3.6.1.5.5.7.48.3) }
```

# 最後に

多過ぎんか...
今回はX509証明書に関して書かれているRFC 5280で定義されている拡張のうち，代表的なものを紹介しました。
CRL拡張や新たに策定されている拡張もあり，全てを網羅することは難しいかもしれません。
しかし，この拡張フィールドがあることで証明書に様々な属性を追加することができ，証明書の柔軟性と機能性が大幅に向上しています。
拡張の理解は，証明書の構造と機能を理解する上で重要な要素となります。
今後，この拡張を使ってやってみたいことを考えているのでお楽しみに。	

# 参考

- RFC 5280([https://datatracker.ietf.org/doc/html/rfc5280](https://datatracker.ietf.org/doc/html/rfc5280)) 
