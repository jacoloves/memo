# AWSモダンアプリケーション入門を読んだメモ

**Date:** 2024-08-04 11:24:55

## まとめ
モダンアプリケーションの`考え方`を学べる本でだった。
手を動かすようなハンズオンではなく、アーキテクチャのお題が出てきて自分の考えを述べるような形式で進んでいく。
お題がありその後著者の解説があるので、自分の考えと著者の考えを比較していくことで学べる。
この本を読んで会社のアーキテクチャの構成理由や実際のアーキテクチャ図を読んだときにその背景を理解するのに役立つのと、
インフラチームのTLとの1on1の会話の時に話されていた内容が多少は理解できるようになった。

## chap0
- モノリシック
  - 1つのアプリケーションに全ての機能を詰め込んだアプリケーション
- マイクロサービス
  - 小さなサービスを組み合わせて機能を提供するアプリケーション
- シラバス的な説明

## chap1
- MVPとは
  - 最小限の機能を持つプロダクト
  - Minimum Viable Product
- モジュラーアーキテクチャ
  - モジュール化されたアーキテクチャ
  - 個別に素早く変更できるから全体のリスクを下げられる
- モダンアプリケーションのベストプラクティス
  - モニタリング
  - サーバーレステクノロジー
  - リリースパイプライン
    - CI/CD
  - モジュラーアーキテクチャ
    - 各機能をモジュール化
    - モノリシック
    - モジュラーモノリス
    - マイクロサービス

## chap2
- 実際の企業にありそうなお題をロールプレイしていくように進めていくみたい
- Sample Book StoreのMVPはかき
  - 本の一覧を表示する
  - 本の検索
  - 会員登録
  - 本の購入
  - 購入した本のPDFをダウンロード
  - 購入履歴の確認
- アーキテクチャ
  - Amazon Route 53
  - Amazon CloudFront
  - Amazon S3
  - Amazon EC2
  - Amazon EC2 Auto Scaling
  - Elastic Load Balancing
  - Amazon RDS
  - Amazon CloudWatch
- アプリケーション構成
  - Ruby on Rails
  - Capistrano
  - Nginx
  - Puma
  - GitHub
  - Jenkins
- ユーザー体験の課題
- システムの課題
- 組織の課題
- 以降の章で課題を解決していく

## chap3
- Twelve-Factor App
  - 12の要素
  - 12の要素を満たすアプリケーションはクラウドネイティブアプリケーションと呼ばれる
  - 12の要素
    - コードベース
    - 依存関係
    - 設定
    - バックエンドサービス
    - ビルド、リリース、実行
    - プロセス
    - ポートバインディング
    - 並行性
    - 廃棄可能性
    - 開発環境と本番環境の一貫性
    - ログ
    - 管理プロセス
- Beyound the Twelve-Factor App
  - 12の要素を超えたアプリケーション
  - モダンアプリケーションのベストプラクティス
- モノリポ
  - 1つのリポジトリに全てのコードを詰め込む
  - マイクロサービスアーキテクチャではない
- APIエンドポイントを環境ごとに分ける
  - ymlファイルで管理
- AWS SDKを本書では使ってるがどうなんだろう
  - 言語ごとに得意不得意があるから新規参画者には難しいかも
  - IaCをしたいならTerraformの方がいいかも
  - HCLに言語統一できるため

## chap4
- コンテナやサーバーレスを使う場合も可視化戦略は必要
- データの取得・活用方法
  - 広告のクリック数
    - クリック数が減っている場合
      - 広告のデザインを変更
      - キャッチコピーを変更
      - ターゲットを変更
  - ユーザーのサイト回遊時間
    - サイト回遊時間が短い場合
      - サイトのデザインを変更
      - サイトの構成を変更
      - サイトのコンテンツを変更
- DevOpsモデルを採用する
  - 開発と運用を一体化
  - 開発と運用の壁を取り払う
  - 開発と運用の連携を強化
  - 開発と運用の責任を共有
- USEモデル
  - Utilization
  - Saturation
  - Errors
- 取得るすべきデータ
  - ビジネスデータ
  - システムデータ
  - 運用データ

## chap5
- サーバレスとコンテナの違い
  - サーバーレス
    - サーバーの管理が不要
    - サーバーのスケールアウトが不要
    - イベント駆動
    - リクエストごとに課金
  - コンテナ
    - サーバーの管理が必要
    - サーバーのスケールアウトが必要
    - コンテナごとに課金

## chap6
- CI/CDについて
  - CI
    - コードの品質を保つ
    - テストを自動化
    - ビルドを自動化
  - CD
    - リリースを自動化
    - デプロイを自動化
    - テストを自動化
- CI/CDパイプラインの構築は開発の初期段階で行う
  - 自動化の恩恵をはじめから受けられる
  - 組織やビズネスの成熟に伴ってCI/CDパイプラインを改善していきやすい
- パイプライン・ファースト
  - パイプラインを最初に構築する
  - パイプラインを最初に構築することで、開発者はコードを書くと同時にCI/CDパイプラインを意識する
- ローリングデプロイメント
  - ローリングデプロイメントは、新しいバージョンのアプリケーションを段階的にデプロイする方法
  - ローリングデプロイメントは、デプロイメントの失敗を最小限に抑える
  - ローリングデプロイメントは、デプロイメントのロールバックを容易にする
- Blue/Greenデプロイメント
  - Blue/Greenデプロイメントは、新しいバージョンのアプリケーションを別の環境にデプロイし、トラフィックを切り替える方法
  - Blue/Greenデプロイメントは、デプロイメントの失敗を最小限に抑える
  - Blue/Greenデプロイメントは、デプロイメントのロールバックを容易にする
- カナリアリリース
  - カナリアリリースは、一部のユーザーに新しいバージョンのアプリケーションをデプロイし、その挙動を監視する方法
  - カナリアリリースは、デプロイメントの失敗を最小限に抑える
  - カナリアリリースは、デプロイメントのロールバックを容易にする

## chap7
- データベースについて
- Pupose-builtデータベース
  - データベースの目的に合わせて選択する
  - データベースの目的に合わせて選択することで、データベースのパフォーマンスを最適化できる
- Evictions
  - キャッシュのエントリーがキャッシュの容量を超えた場合、エントリーを削除すること

## chap8
- 下記のような戦略をとった
  - 非同期な通信、イベント駆動の処理はAWS Lambdaを使う
  - 同期な通信、APIを提供するサービスはAmazon ECSを使う
- BFF
  - フロントエンドとバックエンドの間に配置するサービス
  - フロントエンドとバックエンドの間に配置することで、フロントエンドとバックエンドの間の通信を最適化する
- メッセージングパターン
  - キューモデル
    - キューにメッセージを送信する
    - キューからメッセージを受信する
  - pubsubモデル
    - トピックにメッセージを送信する
    - トピックからメッセージを受信する
- CQRSパターン
  - コマンドとクエリを分離する
  - コマンドとクエリを分離することで、コマンドとクエリの処理を最適化できる
- サーキットブレーカーパターン
  - サーキットブレーカーパターンは、サービス間の通信を監視し、通信が失敗した場合にサービス間の通信を遮断する方法
  - サーキットブレーカーパターンは、サービス間の通信の障害を最小限に抑える
  - サーキットブレーカーパターンは、サービス間の通信の障害を検知し、通信を遮断する
- サービスメッシュ
  - サービスメッシュは、サービス間の通信を制御する方法
  - サービスメッシュは、サービス間の通信を監視し、通信の品質を保つ
  - サービスメッシュは、サービス間の通信を暗号化する
  - Istio
    - サービスメッシュを提供するオープンソースのプロジェクト
  - AWS App Mesh
    - サービスメッシュを提供するAWSのサービス
- サービスディスカバリ
  - サービスディスカバリは、サービス間の通信を最適化する方法
  - サービスディスカバリは、サービス間の通信を監視し、通信の品質を保つ
  - サービスディスカバリは、サービス間の通信を暗号化する
  - AWS Cloud Map
    - サービスディスカバリを提供するAWSのサービス
    - サービスディスカバリは、サービス間の通信を最適化する
- フィーチャーフラグ
  - フィーチャーフラグは、アプリケーションの機能を有効化・無効化する方法
  - フィーチャーフラグは、アプリケーションの機能を段階的にリリースする
  - フィーチャーフラグは、アプリケーションの機能をテストする
  - if文でフィーチャーフラグを判定する
- AWS X-Ray
  - AWS X-Rayは、分散システムのトレーシングを提供するAWSのサービス
  - AWS X-Rayは、分散システムのトレーシングを可視化する
  - AWS X-Rayは、分散システムのトレーシングを分析する