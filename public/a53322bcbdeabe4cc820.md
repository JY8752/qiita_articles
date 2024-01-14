---
title: 'NFT特化flowブロックチェーンに入門するためにcadence言語を学ぶ[Non Fungible Tokens(NFT)編1]'
tags:
  - flow
  - Blockchain
  - solidity
  - AdventCalendar2022
  - cadence
private: false
updated_at: '2022-12-25T10:16:44+09:00'
id: a53322bcbdeabe4cc820
organization_url_name: null
---
この記事は筆者の[ソロ　Advent Calendar 2022](https://qiita.com/advent-calendar/2022/panda) 22日目の記事です。

引き続きflowブロックチェーン上にスマートコントラクトを実装するためのCadenceという言語について公式ドキュメントのチュートリアルをやってみたのでその備忘録です。

今回はいよいよトークンを扱った処理のチュートリアルをやっていきたいと思います。最初はNon-Fungible Token(NFT)のチュートリアルになります！

[NFT特化flowブロックチェーンに入門するためにcadence言語を学ぶ[Hello World編]](https://qiita.com/JY8752/private/3f8c742409598d18630b)
[NFT特化flowブロックチェーンに入門するためにcadence言語を学ぶ[リソース編]](https://qiita.com/JY8752/items/e3027344e5607ee9e7f0)
[NFT特化flowブロックチェーンに入門するためにcadence言語を学ぶ[capability リンクの参照とスクリプト]](https://qiita.com/JY8752/items/3850784684da9b888c05)
[NFT特化flowブロックチェーンに入門するためにcadence言語を学ぶ[Non Fungible Tokens(NFT)編1]](https://qiita.com/JY8752/items/a53322bcbdeabe4cc820) <- 今ここ
[NFT特化flowブロックチェーンに入門するためにcadence言語を学ぶ[Non Fungible Tokens(NFT)編2]](https://qiita.com/JY8752/items/0e2195c296c0f026bcf8)
[NFT特化flowブロックチェーンに入門するためにcadence言語を学ぶ[Fungible Tokens(FT)編]](https://qiita.com/JY8752/items/9760c6238b21e19151a0)

チュートリアルplaygroundはこちら

https://play.onflow.org/a21087ad-b22c-4981-b49e-17297e916fa6

# CadenceにおけるNFT
NFTの詳しい説明はここでは省略しますが、Cadenceでは、他のスマートコントラクト言語とは異なり、アカウント内のストレージにリソースオブジェクトとしてNFTを保存します。このため、NFTはCadenceの型システムによって、単一所有者となることや複製不可であること、偶発的な損失などを防ぐことができ、NFTをアカウント所持者が安全にデジタル資産として所持しているということを認識することができます。

# NFTをアカウントに追加する
チュートリアルplaygroundの```0x01```アカウントのページを開くと以下のようなNFTコントラクトがあるので中身を確認していきましょう。

```typescript:Basic.cdc
pub contract BasicNFT {

    // Declare the NFT resource type
    pub resource NFT {
        // The unique ID that differentiates each NFT
        pub let id: UInt64

        // String mapping to hold metadata
        pub var metadata: {String: String}

        // Initialize both fields in the init function
        init(initID: UInt64) {
            self.id = initID
            self.metadata = {}
        }
    }

    // Function to create a new NFT
    pub fun createNFT(id: UInt64): @NFT {
        return <-create NFT(initID: id)
    }

    // Create a single new NFT and save it to account storage
    init() {
        self.account.save<@NFT>(<-create NFT(initID: 1), to: /storage/BasicNFTPath)
    }
}
```
前回までに学んだ内容で理解することができますが、アカウント内に```BasicNFT```という名前でコントラクトを定義し、その内部で```NFT```という名前のリソースを宣言しています。リソースには整数値の```id```とmapで```metadata```をフィールドとして持つようにし、initでフィールドの初期化をしています。

NFTリソースの作成には```createNFT```関数を定義していて、コントラクトの初期化処理で自信のアカウントストレージに作成したNFTを保存しています。

NFTが保存できているかを確認するために、このコントラクトをデプロイし、NFT Existsトランザクションを実行してみます。トランザクションの中身は以下のようになっています。

```typescript
// NFT Exists

import BasicNFT from 0x01

// This transaction checks if an NFT exists in the storage of the given account
// by trying to borrow from it. If the borrow succeeds (returns a non-nil value), the token exists!
transaction {
    prepare(acct: AuthAccount) {
        if acct.borrow<&BasicNFT.NFT>(from: /storage/BasicNFTPath) != nil {
            log("The token exists!")
        } else {
            log("No token found!")
        }
    }
}
```

ここでは、前回の記事で紹介した**capability**は使用せず、アカウントストレージから直接NFTリソースへの参照を借用し、存在していれば成功のログを出すような処理をしています。こちらを実行してみると以下のように、ちゃんとNFTが存在していることがわかります。

```
NFT Exists "The token exists!"
```

# NFTの移動
別のアカウントにNFTを移動させる方法についてですが、チュートリアルに「ここは学習のために自分で考えてみてね」とあるので自力でやってみる。まず、playgroundのBasic Transferを開くと以下のようになっている。

```typescript
import BasicNFT from 0x01

/// Basic transaction for two accounts to authorize
/// to transfer an NFT

transaction {
    prepare(signer1: AuthAccount, signer2: AuthAccount) {

        // Fill in code here to load the NFT from signer1
        // and save it into signer2's storage
    }
}
```

prepareブロックの中を埋めてみる

```diff_typescript
import BasicNFT from 0x01

/// Basic transaction for two accounts to authorize
/// to transfer an NFT

transaction {
    prepare(signer1: AuthAccount, signer2: AuthAccount) {

        // Fill in code here to load the NFT from signer1
        // and save it into signer2's storage
+        let nftResource <- signer1.load<@BasicNFT.NFT>(from: /storage/BasicNFTPath)
+        signer2.save(<- nftResource, to: /storage/BasicNFTPath)

+        log(signer2.borrow<&BasicNFT.NFT>(from: /storage/BasicNFTPath) != nil)
    }
}
```
signer1からNFTを読み込んで、signer2に保存して、signer2のストレージを参照してみてログに出してみる。保存されていればtrueが出力されるはず。実行してみる

```
Basic Transfer Error failed to force-cast value: expected type `BasicNFT.NFT`, got `BasicNFT.NFT?`
```

エラーになった。読み込んだリソースがオプショナルなのを考慮していないためエラーになっているっぽい。以下のように修正

```typescript
let nftResource <- signer1.load<@BasicNFT.NFT>(from: /storage/BasicNFTPath)
 ?? panic("Could not load NFT")
```

実行してみる
```
Basic Transfer true
```

無事、NFTを移動させることができた

# おまけ Solidityとの比較
チュートリアルで作成したBasic.cdcの内容をSolidityで書いてみました。Solidityはまだ学習中なので誤った点があればコメントください。

```typescript
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

contract BasicNFT {
  uint private tokenId;
  mapping(address => uint[]) private ownedNFTMapping;

  constructor() {
    ownedNFTMapping[msg.sender].push(1);
    tokenId = 1;
  }

  function createNFT() public {
    tokenId++;
    ownedNFTMapping[msg.sender].push(tokenId);
  }

  function getId() public view returns (uint[] memory) {
    return ownedNFTMapping[msg.sender];
  }
}
```

なんとなく似たようなことをSolidityでやろうとするとこうなる。Cadenceとの大きな違いが

```typescript
mapping(address => uint[]) private ownedNFTMapping;
```

この部分でSolidityでアカウントがNFTを所持するようにするにはmapでアカウントアドレスに対してNFT情報を紐付けコントラクトのフィールドで記録するような実装をする必要があり中央台帳的な管理になります。

CadenceではアカウントのストレージにNFTリソースを保存することができるので現実世界の人がものを所有するということをコードで表現できてるような気がします。

# まとめ
今回は以下のことについて紹介しました。
- NFTをアカウントストレージ内に保存する方法について
- アカウントのNFTを別のアカウントに移動する方法について

感覚的な話ですがアカウントのストレージ内にNFTリソースが保存されているのが非常にわかりやすいというか**所持してるんだな**という気がしました。まだ、実際に出回っているNFTをどう実装するのかについての理解が足りないので引き続き次回もNFTについて学んでいきたいと思います！今回は以上です！
