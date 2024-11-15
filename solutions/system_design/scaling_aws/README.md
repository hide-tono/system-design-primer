# AWSで数百万人のユーザーにスケールするシステムを設計する

*注: このドキュメントは重複を避けるために[システム設計トピック](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics)の関連部分に直接リンクしています。一般的な話題、トレードオフ、および代替案についてはリンク先の内容を参照してください。*

## ステップ1: ユースケースと制約を概説する

> 要件を収集し、問題の範囲を決定します。
> ユースケースと制約を明確にするために質問をします。
> 仮定について議論します。

面接官に明確化の質問をすることができない場合、いくつかのユースケースと制約を定義します。

### ユースケース

この問題を解決するには、1) **ベンチマーク/負荷テスト**、2) ボトルネックの**プロファイル**、3) ボトルネックに対処しながら代替案とトレードオフを評価し、4) 繰り返すという反復的なアプローチが必要です。これは基本設計をスケーラブルな設計に進化させるための良いパターンです。

AWSの背景があるか、AWSの知識が必要なポジションに応募している場合を除き、AWS固有の詳細は必須ではありません。ただし、この演習で議論される原則の多くはAWSエコシステム外でも一般的に適用できます。

#### 以下のユースケースに限定して問題の範囲を定義します

* **ユーザー**が読み取りまたは書き込みリクエストを行う
    * **サービス**が処理を行い、ユーザーデータを保存し、結果を返す
* **サービス**が少数のユーザーから数百万人のユーザーに対応するように進化する必要がある
    * 多数のユーザーとリクエストを処理するためにアーキテクチャを進化させる際の一般的なスケーリングパターンについて議論する
* **サービス**が高可用性を持つ

### 制約と仮定

#### 仮定を述べる

* トラフィックは均等に分散されていない
* リレーショナルデータが必要
* 1ユーザーから数千万ユーザーにスケールする必要がある
    * ユーザーの増加を次のように表す:
        * Users+
        * Users++
        * Users+++
        * ...
    * 1000万ユーザー
    * 月間10億回の書き込み
    * 月間1000億回の読み取り
    * 読み取りと書き込みの比率は100:1
    * 書き込みごとに1KBのコンテンツ

#### 使用量を計算する

**面接官にバックオブエンベロープの使用量計算を行うべきか確認してください。**

* 月間1TBの新しいコンテンツ
    * 書き込みごとに1KB * 月間10億回の書き込み
    * 3年間で36TBの新しいコンテンツ
    * ほとんどの書き込みは既存のコンテンツの更新ではなく新しいコンテンツからのものであると仮定
* 平均で毎秒400回の書き込み
* 平均で毎秒40,000回の読み取り

便利な変換ガイド:

* 月間250万秒
* 毎秒1リクエスト = 月間250万リクエスト
* 毎秒40リクエスト = 月間1億リクエスト
* 毎秒400リクエスト = 月間10億リクエスト

## ステップ2: 高レベルの設計を作成する

> 重要なコンポーネントをすべて含む高レベルの設計を概説します。

![Imgur](http://i.imgur.com/B8LDKD7.png)

## ステップ3: コアコンポーネントを設計する

> 各コアコンポーネントの詳細に入ります。

### ユースケース: ユーザーが読み取りまたは書き込みリクエストを行う

#### 目標

* ユーザーが1〜2人しかいない場合、基本的なセットアップのみが必要
    * 単一のボックスでシンプルに
    * 必要に応じて垂直スケーリング
    * ボトルネックを特定するために監視

#### 単一のボックスから始める

* **EC2上のWebサーバー**
    * ユーザーデータのストレージ
    * [**MySQLデータベース**](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)

**垂直スケーリング**を使用:

* 単に大きなボックスを選ぶ
* スケールアップの方法を決定するためにメトリクスを監視
    * ボトルネックを特定するために基本的な監視を使用: CPU、メモリ、IO、ネットワークなど
    * CloudWatch、top、nagios、statsd、graphiteなど
* 垂直スケーリングは非常に高価になる可能性がある
* 冗長性/フェイルオーバーがない

*トレードオフ、代替案、および追加の詳細:*

* **垂直スケーリング**の代替案は[**水平スケーリング**](https://github.com/donnemartin/system-design-primer#horizontal-scaling)

#### SQLから始めて、NoSQLを検討する

制約はリレーショナルデータが必要であることを仮定しています。単一のボックスで**MySQLデータベース**を使用して開始できます。

*トレードオフ、代替案、および追加の詳細:*

* [リレーショナルデータベース管理システム (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)セクションを参照
* [SQLまたはNoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)を使用する理由について議論する

#### 公開静的IPを割り当てる

* Elastic IPは再起動時にIPが変わらない公開エンドポイントを提供
* フェイルオーバーに役立ち、ドメインを新しいIPにポイントするだけで済む

#### DNSを使用する

インスタンスの公開IPにドメインをマップするためにRoute 53などの**DNS**を追加します。

*トレードオフ、代替案、および追加の詳細:*

* [ドメインネームシステム](https://github.com/donnemartin/system-design-primer#domain-name-system)セクションを参照

#### Webサーバーを保護する

* 必要なポートのみを開放する
    * Webサーバーが応答する必要がある受信リクエスト:
        * HTTP用の80
        * HTTPS用の443
        * ホワイトリストに登録されたIPのみのSSH用の22
    * Webサーバーが外部への接続を開始するのを防ぐ

*トレードオフ、代替案、および追加の詳細:*

* [セキュリティ](https://github.com/donnemartin/system-design-primer#security)セクションを参照

## ステップ4: 設計をスケールする

> 制約を考慮してボトルネックを特定し、対処します。

### Users+

![Imgur](http://i.imgur.com/rrfjMXB.png)

#### 仮定

ユーザー数が増加し、単一のボックスの負荷が増加しています。**ベンチマーク/負荷テスト**と**プロファイリング**により、**MySQLデータベース**がますます多くのメモリとCPUリソースを消費し、ユーザーコンテンツがディスクスペースを埋めていることが示されています。

これまで**垂直スケーリング**でこれらの問題に対処してきましたが、非常に高価になり、**MySQLデータベース**と**Webサーバー**の独立したスケーリングができません。

#### 目標

* 単一のボックスの負荷を軽減し、独立したスケーリングを可能にする
    * 静的コンテンツを**オブジェクトストア**に別々に保存する
    * **MySQLデータベース**を別のボックスに移動する
* 欠点
    * これらの変更は複雑さを増し、**Webサーバー**を**オブジェクトストア**および**MySQLデータベース**にポイントするように変更する必要がある
    * 新しいコンポーネントを保護するために追加のセキュリティ対策が必要
    * AWSのコストも増加する可能性があるが、同様のシステムを自分で管理するコストと比較する必要がある

#### 静的コンテンツを別々に保存する

* 静的コンテンツを保存するためにS3のような管理された**オブジェクトストア**を使用することを検討する
    * 非常にスケーラブルで信頼性が高い
    * サーバーサイド暗号化
* 静的コンテンツをS3に移動する
    * ユーザーファイル
    * JS
    * CSS
    * 画像
    * ビデオ

#### MySQLデータベースを別のボックスに移動する

* **MySQLデータベース**を管理するためにRDSのようなサービスを使用することを検討する
    * 管理が簡単でスケールしやすい
    * 複数の可用性ゾーン
    * 保存時の暗号化

#### システムを保護する

* データを転送中および保存時に暗号化する
* 仮想プライベートクラウドを使用する
    * インターネットからのトラフィックを送受信できるように単一の**Webサーバー**用のパブリックサブネットを作成する
    * 他のすべてのもののためにプライベートサブネットを作成し、外部アクセスを防ぐ
    * 各コンポーネントのホワイトリストに登録されたIPからのみポートを開放する
* これらの同じパターンは、演習の残りの新しいコンポーネントにも実装する必要がある

*トレードオフ、代替案、および追加の詳細:*

* [セキュリティ](https://github.com/donnemartin/system-design-primer#security)セクションを参照

### Users++

![Imgur](http://i.imgur.com/raoFTXM.png)

#### 仮定

**ベンチマーク/負荷テスト**と**プロファイリング**により、ピーク時に単一の**Webサーバー**がボトルネックとなり、応答が遅くなり、場合によってはダウンタイムが発生することが示されています。サービスが成熟するにつれて、より高い可用性と冗長性に��かって進みたいと考えています。

#### 目標

* 次の目標は**Webサーバー**のスケーリング問題に対処することを目的としています
    * **ベンチマーク/負荷テスト**と**プロファイリング**に基づいて、これらの技術のうち1つまたは2つだけを実装する必要があるかもしれません
* 増加する負荷に対応し、単一障害点に対処するために[**水平スケーリング**](https://github.com/donnemartin/system-design-primer#horizontal-scaling)を使用する
    * AmazonのELBやHAProxyなどの[**ロードバランサー**](https://github.com/donnemartin/system-design-primer#load-balancer)を追加する
        * ELBは高可用性
        * 自分で**ロードバランサー**を設定する場合、複数の可用性ゾーンで[アクティブ-アクティブ](https://github.com/donnemartin/system-design-primer#active-active)または[アクティブ-パッシブ](https://github.com/donnemartin/system-design-primer#active-passive)の複数のサーバーを設定すると可用性が向上します
        * SSLを**ロードバランサー**で終了させることで、バックエンドサーバーの計算負荷を軽減し、証明書管理を簡素化します
    * 複数の**Webサーバー**を複数の可用性ゾーンに分散して使用する
    * 複数の**MySQL**インスタンスを複数の可用性ゾーンにわたる[**マスター-スレーブフェイルオーバー**](https://github.com/donnemartin/system-design-primer#master-slave-replication)モードで使用して冗長性を向上させる
* **Webサーバー**を[**アプリケーションサーバー**](https://github.com/donnemartin/system-design-primer#application-layer)から分離する
    * 両方のレイヤーを独立してスケールおよび構成する
    * **Webサーバー**は[**リバースプロキシ**](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)として動作できます
    * 例えば、**Read API**を処理する**アプリケーションサーバー**を追加し、他の**Write API**を処理することができます
* 静的（および一部の動的）コンテンツをCloudFrontなどの[**コンテンツデリバリネットワーク（CDN）**](https://github.com/donnemartin/system-design-primer#content-delivery-network)に移動して負荷と遅延を減らす

*トレードオフ、代替案、および追加の詳細:*

* 詳細については上記のリンク先を参照してください

### Users+++

![Imgur](http://i.imgur.com/OZCxJr0.png)

**注:** **内部ロードバランサー**は混乱を避けるために表示されていません

#### 仮定

**ベンチマーク/負荷テスト**と**プロファイリング**により、読み取りが多く（書き込みに対して100:1）データベースが高い読み取り要求によるパフォーマンスの低下に苦しんでいることが示されています。

#### 目標

* 次の目標は**MySQLデータベース**のスケーリング問題に対処することを目的としています
    * **ベンチマーク/負荷テスト**と**プロファイリング**に基づいて、これらの技術のうち1つまたは2つだけを実装する必要があるかもしれません
* 負荷と遅延を減らすために、次のデータをElasticacheなどの[**メモリキャッシュ**](https://github.com/donnemartin/system-design-primer#cache)に移動する:
    * **MySQL**から頻繁にアクセスされるコンテンツ
        * 最初に**MySQLデータベース**キャッシュを構成して、それがボトルネックを解消するのに十分かどうかを確認してから**メモリキャッシュ**を実装する
    * **Webサーバー**からのセッションデータ
        * **Webサーバー**がステートレスになり、**オートスケーリング**が可能になる
    * メモリから1MBを順次読み取るのに約250マイクロ秒かかりますが、SSDからの読み取りは4倍、ディスクからの読み取りは80倍長くかかります。<sup><a href=https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know>1</a></sup>
* 書き込みマスターの負荷を軽減するために[**MySQLリードレプリカ**](https://github.com/donnemartin/system-design-primer#master-slave-replication)を追加する
* 応答性を向上させるために、より多くの**Webサーバー**および**アプリケーションサーバー**を追加する

*トレードオフ、代替案、および追加の詳細:*

* 詳細については上記のリンク先を参照してください

#### MySQLリードレプリカを追加する

* **メモリキャッシュ**を追加およびスケーリングすることに加えて、**MySQLリードレプリカ**も**MySQL書き込みマスター**の負荷を軽減するのに役立ちます
* 書き込みと読み取りを分離するためのロジックを**Webサーバー**に追加する
* **MySQLリードレプリカ**の前に**ロードバランサー**を追加する（混乱を避けるために表示されていません）
* ほとんどのサービスは書き込みよりも読み取りが多い

*トレードオフ、代替案、および追加の詳細:*

* [リレーショナルデータベース管理システム (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)セクションを参照

### Users++++

![Imgur](http://i.imgur.com/3X8nmdL.png)

#### 仮定

**ベンチマーク/負荷テスト**と**プロファイリング**により、米国の通常の営業時間中にトラフィックが急増し、ユーザーがオフィスを離れると大幅に減少することが示されています。実際の負荷に基づいてサーバーを自動的にスピンアップおよびスピンダウンすることでコストを削減できると考えています。私たちは小さなショップなので、**オートスケーリング**および一般的な運用のためにできるだけ多くのDevOpsを自動化したいと考えています。

#### 目標

* 必要に応じて容量をプロビジョニングするために**オートスケーリング**を追加する
    * トラフィックの急増に対応する
    * 使用されていないインスタンスの電源を切ることでコストを削減する
* DevOpsを自動化する
    * Chef、Puppet、Ansibleなど
* ボトルネックに対処するためにメトリクスの監視を続ける
    * **ホストレベル** - 単一のEC2インスタンスをレビューする
    * **集計レベル** - ロードバランサーの統計をレビューする
    * **ログ分析** - CloudWatch、CloudTrail、Loggly、Splunk、Sumo
    * **外部サイトのパフォーマンス** - PingdomまたはNew Relic
    * **通知とインシデントの処理** - PagerDuty
    * **エラーレポート** - Sentry

#### オートスケーリングを追加する

* AWSの**オートスケーリング**などの管理サービスを検討する
    * 各**Webサーバー**および各**アプリケーションサーバー**タイプごとに1つのグループを作成し、各グループを複数の可用性ゾーンに配置する
    * インスタンスの最小数と最大数を設定する
    * CloudWatchを通じてスケールアップおよびスケールダウンをトリガーする
        * 予測可能な負荷のための単純な時間帯メトリックまたは
        * 一定期間のメトリック:
            * CPU負荷
            * レイテンシ
            * ネットワークトラフィック
            * カスタムメトリック
    * 欠点
        * オートスケーリングは複雑さを増す可能性がある
        * システムが増加した需要に対応するために適切にスケールアップするまで、または需要が減少したときにスケールダウンするまでに時間がかかる場合があります

### Users+++++

![Imgur](http://i.imgur.com/jj3A5N8.png)

**注:** 混乱を避けるために**オートスケーリング**グループは表示されていません

#### 仮定

サービスが制約で概説された数値に向かって成長し続けるにつれて、新しいボトルネックを発見し対処するために**ベンチマーク/負荷テスト**と**プロファイリング**を反復的に実行します。

#### 目標

問題の制約によるスケーリング問題に対処し続けます:

* **MySQLデータベース**が大きくなりすぎる場合、データベースに限定された期間のデータのみを保存し、残りをRedshiftなどのデータウェアハウスに保存することを検討するかもしれません
    * Redshiftのようなデータウェアハウスは、月間1TBの新しいコンテンツの制約を快適に処理できます
* 平均で毎秒40,000回の読み取り要求があるため、人気のあるコンテンツの読み取りトラフィックは**メモリキャッシュ**をスケーリングすることで対処でき、不均等に分散されたトラフィックやトラフィックの急増にも対応できます
    * **SQLリードレプリカ**はキャッシュミスの処理に苦労する可能性があり、追加のSQLスケーリングパターンを採用する必要があるかもしれません
* 平均で毎秒400回の書き込み（おそらくピーク時にはそれ以上）が単一の**SQL書き込みマスター-スレーブ**にとって厳しいかもしれず、追加のスケーリング技術が必要になることを示しています

SQLスケーリングパターンには次のものがあります:

* [フェデレーション](https://github.com/donnemartin/system-design-primer#federation)
* [シャーディング](https://github.com/donnemartin/system-design-primer#sharding)
* [非正規化](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQLチューニング](https://github.com/donnemartin/system-design-primer#sql-tuning)

高い読み取りおよび書き込み要求にさらに対処するために、DynamoDBなどの[**NoSQLデータベース**](https://github.com/donnemartin/system-design-primer#nosql)に適切なデータを移動することも検討する必要があります。

[**アプリケーションサーバー**](https://github.com/donnemartin/system-design-primer#application-layer)をさらに分離して独立したスケーリングを可能にすることができます。リアルタイムで行う必要のないバッチ処理や計算は、**キュー**と**ワーカー**を使用して[**非同期**](https://github.com/donnemartin/system-design-primer#asynchronism)に行うことができます:

* 例えば、写真サービスでは、写真のアップロードとサムネイルの作成を分離できます:
    * **クライアント**が写真をアップロードする
    * **アプリケーションサーバー**がSQSなどの**キュー**にジョブを入れる
    * EC2またはLambda上の**ワーカーサービス**が**キュー**から作業を引き出し、その後:
        * サムネイルを作成する
        * **データベース**を更新する
        * サムネイルを**オブジェクトストア**に保存する

*トレードオフ、代替案、および追加の詳細:*

* 詳細については上記のリンク先を参照してください

## 追加の話題

> 問題の範囲と残り時間に応じて、掘り下げる追加のトピック。

### SQLスケーリングパターン

* [リードレプリカ](https://github.com/donnemartin/system-design-primer#master-slave-replication)
* [フェデレーション](https://github.com/donnemartin/system-design-primer#federation)
* [シャーディング](https://github.com/donnemartin/system-design-primer#sharding)
* [非正規化](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQLチューニング](https://github.com/donnemartin/system-design-primer#sql-tuning)

#### NoSQL

* [キー-バリューストア](https://github.com/donnemartin/system-design-primer#key-value-store)
* [ドキュメントストア](https://github.com/donnemartin/system-design-primer#document-store)
* [ワイドカラムストア](https://github.com/donnemartin/system-design-primer#wide-column-store)
* [グラフデータベース](https://github.com/donnemartin/system-design-primer#graph-database)
* [SQL対NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)

### キャッシュ

* どこにキャッシュするか
    * [クライアントキャッシュ](https://github.com/donnemartin/system-design-primer#client-caching)
    * [CDNキャッシュ](https://github.com/donnemartin/system-design-primer#cdn-caching)
    * [Webサーバーキャッシュ](https://github.com/donnemartin/system-design-primer#web-server-caching)
    * [データベースキャッシュ](https://github.com/donnemartin/system-design-primer#database-caching)
    * [アプリケーションキャッシュ](https://github.com/donnemartin/system-design-primer#application-caching)
* 何をキャッシュするか
    * [データベースクエリレベルでのキャッシュ](https://github.com/donnemartin/system-design-primer#caching-at-the-database-query-level)
    * [オブジェクトレベルでのキャッシュ](https://github.com/donnemartin/system-design-primer#caching-at-the-object-level)
* いつキャッシュを更新するか
    * [キャッシュアサイド](https://github.com/donnemartin/system-design-primer#cache-aside)
    * [ライトスルー](https://github.com/donnemartin/system-design-primer#write-through)
    * [ライトビハインド（ライトバック）](https://github.com/donnemartin/system-design-primer#write-behind-write-back)
    * [リフレッシュアヘッド](https://github.com/donnemartin/system-design-primer#refresh-ahead)

### 非同期処理とマイクロサービス

* [メッセージキュー](https://github.com/donnemartin/system-design-primer#message-queues)
* [タスクキュー](https://github.com/donnemartin/system-design-primer#task-queues)
* [バックプレッシャー](https://github.com/donnemartin/system-design-primer#back-pressure)
* [マイクロサービス](https://github.com/donnemartin/system-design-primer#microservices)

### 通信

* トレードオフを議論する:
    * クライアントとの外部通信 - [RESTに従ったHTTP API](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)
    * 内部通信 - [RPC](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)
* [サービスディスカバリー](https://github.com/donnemartin/system-design-primer#service-discovery)

### セキュリティ

[セキュリティセクション](https://github.com/donnemartin/system-design-primer#security)を参照してください。

### レイテンシ数値

[プログラマーが知っておくべきレイテンシ数値](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know)を参照してください。

### 継続的な取り組み

* ボトルネックに対処するためにシステムのベンチマークと監視を続ける
* スケーリングは反復的なプロセスです
