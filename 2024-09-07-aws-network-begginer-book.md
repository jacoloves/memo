# AWSネットワーク入門 第2版の所感

**Date:** 2024-09-07 18:20:30

[AWSネットワーク入門 第2版](https://book.impress.co.jp/books/1122101027)を読んでAWSのネットワークについて学んだので、
各章で学んだことや所感を書こうと思います。

## 所感
AWSのネットワークの基礎を学ぶにはいい本だと思います。
1~6章を読んでハンズオンをやるとWordPressが動かせるまで構築が完了するので、他のアプリにも応用ができそうです。
インフラエンジニアやクラウドエンジニアは7,8章も併せて読んでおくてAWSおよびドメイン周りの理解が深まります。
入門と書いてある通り業務で使うような突っ込んだ知識は得にくいのかなと感じました。特に7章のALBやRoute53周りで[加重ルーティング](https://docs.aws.amazon.com/ja_jp/Route53/latest/DeveloperGuide/routing-policy-weighted.html)の説明は欲しかったですが入門書などのそこまで深い知識はいらないですね。
業務でインフラ周りを担当しているエンジニアはAWSのネットワークについてさらに突っ込んだ本を読むか公式のハンズオンを実施する方が知識が身につくのかなという所感でした。

## 1章
AWSとオンプレの比較やネットワークの基礎知識を学びました。
ネットワークの本に書いてあることがつらつらと書いてありました。
データセンターを例にAWSのネットワークを説明しててわかりやすかったです。
グローバル、リージョン、アベイラビリティゾーンサービスの区別を覚えました。（今までなんとなくで覚えてました。
今までAWSのアーキテクチャ図を軽くではありますが読んできたので、本に書いてある簡易なアーキテクチャは苦も無く理解できました。

## 2章
VPCについて学びました。
実際に手を動かしてVPCやサブネットを構築するハンズオンを実施したり、CIDRなどの座学をざっくりと学びました。
デフォルトVPCは削除しても再作成が可能だとわかりました。

## 3章
EC2やEBSについて学びました。
T,M,C,X,R,G,I,Dシリーズの用途を知れて勉強になりました。
ハンズオンでEC2を構築して、ENIの確認などをしました。

## 4章
パブリックIPアドレス割り当て→IGW設定→ルートテーブル設定→Elastic IPの割り当てまで行いました。
あまりElastic IPの割り当てまでは実施しないので勉強になりました。
AWSネットワーク上にはメタデータを配信するHTTPサーバーが存在するそうです。（http://169.254.169.254）
(https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html)

## 5章
セキュリティグループとネットワークACLについて学びました。
セキュリティグループ：ENI単位で設定するパケットフィルタリング機能
ネットワークACL：サブネット単位で設定するパケットフィルタリング機能
Appacheサーバーを導入して、セキュリティグループのインバウンドに HTTPとHTTPSを追加して接続できることを確認しました。

## 6章
プライベートサブネットのDBサーバーをNATゲートウェイから通信させました。
DBサーバーにMariaDBをインストールしてWordPressを起動するチュートリアルがありましたが、既知の情報だったので飛ばしました。
Elastic IPやNATゲートウェイはお金がかかるので閉じ忘れないように注意しないといけないです。

## 7章
Route53とELBについて学びました。
Route53を使用してドメインを取得していましたが、その部分はスキップしています。
ELBはALBを使用してwebserverを2台に冗長化させて構築しました。

## 8章
S3などの非VPCサービスを接続する方法として、インターフェイスエンドポイントとゲートウェイエンドポイントを構築して通信しました。
後半はVPCピアリングや外部ネットワークとの接続方法を書いてありましたが、ハンズオンは実施しませんでした。
