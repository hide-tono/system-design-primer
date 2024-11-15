# Pastebin.com（またはBit.ly）の設計

*注: このドキュメントは、重複を避けるために[システム設計トピック](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics)の関連部分に直接リンクしています。一般的な話題、トレードオフ、および代替案についてはリンク先の内容を参照してください。*

**Bit.lyの設計** - これは類似ｌ．、；・￥の質問ですが、pastebinは元の未短縮URLの代わりにペースト内容を保存する必要があります。

## ステップ1: ユースケースと制約の概要

> 要件を収集し、問題の範囲を決定します。
> ユースケースと制約を明確にするために質問します。
> 仮定を議論します。

インタビュアーがいない場合、いくつかのユースケースと制約を定義します。

### ユースケース

#### 以下のユースケースのみを扱うように問題の範囲を設定します

* **ユーザー**がテキストブロックを入力し、ランダムに生成されたリンクを取得する
    * 有効期限
        * デフォルト設定では期限切れにならない
        * オプションで時間制限を設定できる
* **ユーザー**がペーストのURLを入力して内容を表示する
* **ユーザー**は匿名
* **サービス**がページの分析を追跡する
    * 月間訪問統計
* **サービス**が期限切れのペーストを削除する
* **サービス**が高可用性を持つ

#### 範囲外

* **ユーザー**がアカウントを登録する
    * **ユーザー**がメールを確認する
* **ユーザー**が登録済みアカウントにログインする
    * **ユーザー**がドキュメントを編集する
* **ユーザー**が可視性を設定する
* **ユーザー**がショートリンクを設定する

### 制約と仮定

#### 仮定を述べる

* トラフィックは均等に分配されていない
* ショートリンクをフォローするのは速くなければならない
* ペーストはテキストのみ
* ページビュー分析はリアルタイムである必要はない
* 1000万人のユーザー
* 月間1000万のペースト書き込み
* 月間1億のペースト読み取り
* 読み取りと書き込みの比率は10:1

#### 使用量を計算する

**インタビュアーにバックオブエンベロープの使用量計算を行うべきか確認してください。**

* ペーストごとのサイズ
    * ペースト内容は1 KB
    * `shortlink` - 7バイト
    * `expiration_length_in_minutes` - 4バイト
    * `created_at` - 5バイト
    * `paste_path` - 255バイト
    * 合計 = 約1.27 KB
* 月間12.7 GBの新しいペースト内容
    * 1.27 KBのペースト * 月間1000万ペースト
    * 3年間で約450 GBの新しいペースト内容
    * 3年間で3億6000万のショートリンク
    * ほとんどが新しいペーストであり、既存のものの更新ではないと仮定
* 平均で1秒あたり4回のペースト書き込み
* 平均で1秒あたり40回の読み取りリクエスト

便利な変換ガイド:

* 月間250万秒
* 1秒あたり1リクエスト = 月間250万リクエスト
* 1秒あたり40リクエスト = 月間1億リクエスト
* 1秒あたり400リクエスト = 月間10億リクエスト

## ステップ2: 高レベルの設計を作成する

> 重要なコンポーネントをすべて含む高レベルの設計を概説します。

![Imgur](http://i.imgur.com/BKsBnmG.png)

## ステップ3: コアコンポーネントを設計する

> 各コアコンポーネントの詳細に入ります。

### ユースケース: ユーザーがテキストブロックを入力し、ランダムに生成されたリンクを取得する

[リレーショナルデータベース](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)を大きなハッシュテーブルとして使用し、生成されたURLをファイルサーバーとペーストファイルを含むパスにマッピングすることができます。

ファイルサーバーを管理する代わりに、Amazon S3のような管理された**オブジェクトストア**や[NoSQLドキュメントストア](https://github.com/donnemartin/system-design-primer#document-store)を使用することができます。

大きなハッシュテーブルとして機能するリレーショナルデータベースの代わりに、[NoSQLキー値ストア](https://github.com/donnemartin/system-design-primer#key-value-store)を使用することもできます。SQLとNoSQLの選択に関する[トレードオフ](https://github.com/donnemartin/system-design-primer#sql-or-nosql)を議論する必要があります。以下の議論ではリレーショナルデータベースアプローチを使用します。

* **クライアント**が**Webサーバー**にペースト作成リクエストを送信します。Webサーバーは[リバースプロキシ](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)として動作します。
* **Webサーバー**がリクエストを**書き込みAPI**サーバーに転送します。
* **書き込みAPI**サーバーは以下を行います:
    * 一意のURLを生成します
        * **SQLデータベース**で重複を確認してURLが一意であることを確認します
        * URLが一意でない場合、別のURLを生成します
        * カスタムURLをサポートする場合、ユーザーが提供したURLを使用します（重複も確認）
    * **SQLデータベース**の`pastes`テーブルに保存します
    * ペーストデータを**オブジェクトストア**に保存します
    * URLを返します

**インタビュアーにどれだけのコードを書くべきか確認してください**。

`pastes`テーブルは以下の構造を持つことができます:

```
shortlink char(7) NOT NULL
expiration_length_in_minutes int NOT NULL
created_at datetime NOT NULL
paste_path varchar(255) NOT NULL
PRIMARY KEY(shortlink)
```

`shortlink`列に基づいて主キーを設定することで、データベースは一意性を強制するための[インデックス](https://github.com/donnemartin/system-design-primer#use-good-indices)を作成します。`created_at`に追加のインデックスを作成して、ルックアップを高速化し（全テーブルをスキャンする代わりにログ時間）、データをメモリに保持します。メモリから1 MBを順次読み取るのに約250マイクロ秒かかりますが、SSDから読み取ると4倍、ディスクから読み取ると80倍長くかかります。<sup><a href=https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know>1</a></sup>

一意のURLを生成するために、以下を行うことができます:

* ユーザーのip_address + タイムスタンプの[**MD5**](https://en.wikipedia.org/wiki/MD5)ハッシュを取ります
    * MD5は128ビットのハッシュ値を生成する広く使用されているハッシュ関数です
    * MD5は均等に分布します
    * 代わりにランダムに生成されたデータのMD5ハッシュを取ることもできます
* MD5ハッシュを[**Base 62**](https://www.kerstner.at/2012/07/shortening-strings-using-base-62-encoding/)でエンコードします
    * Base 62は`[a-zA-Z0-9]`にエンコードされ、URLに適しており、特殊文字のエスケープが不要です
    * 元の入力に対して1つのハッシュ結果しかなく、Base 62は決定論的です（ランダム性は関与しません）
    * Base 64も人気のあるエンコーディングですが、追加の`+`と`/`文字のためURLには問題があります
    * 以下の[Base 62擬似コード](http://stackoverflow.com/questions/742013/how-to-code-a-url-shortener)は、桁数kに対してO(k)時間で実行されます:

```python
def base_encode(num, base=62):
    digits = []
    while num > 0:
      remainder = modulo(num, base)
      digits.push(remainder)
      num = divide(num, base)
    digits = digits.reverse
```

* 出力の最初の7文字を取り、62^7の可能な値を得て、3年間で3億6000万のショートリンクの制約を処理するのに十分です:

```python
url = base_encode(md5(ip_address+timestamp))[:URL_LENGTH]
```

公開[**REST API**](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)を使用します:

```
$ curl -X POST --data '{ "expiration_length_in_minutes": "60", \
    "paste_contents": "Hello World!" }' https://pastebin.com/api/v1/paste
```

レスポンス:

```
{
    "shortlink": "foobar"
}
```

内部通信には[リモートプロシージャコール](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)を使用できます。

### ユースケース: ユーザーがペーストのURLを入力して内容を表示する

* **クライアント**が**Webサーバー**にペースト取得リクエストを送信します
* **Webサーバー**がリクエストを**読み取りAPI**サーバーに転送します
* **読み取りAPI**サーバーは以下を行います:
    * **SQLデータベース**で生成されたURLを確認します
        * URLが**SQLデータベース**にある場合、**オブジェクトストア**からペースト内容を取得します
        * そうでない場合、ユーザーにエラーメッセージを返します

REST API:

```
$ curl https://pastebin.com/api/v1/paste?shortlink=foobar
```

レスポンス:

```
{
    "paste_contents": "Hello World"
    "created_at": "YYYY-MM-DD HH:MM:SS"
    "expiration_length_in_minutes": "60"
}
```

### ユースケース: サービスがページの分析を追跡する

リアルタイム分析が要件でないため、**Webサーバー**のログを**MapReduce**してヒット数を生成するだけで済みます。

**インタビュアーにどれだけのコードを書くべきか確認してください**。

```python
class HitCounts(MRJob):

    def extract_url(self, line):
        """ログ行から生成されたURLを抽出します。"""
        ...

    def extract_year_month(self, line):
        """タイムスタンプから年と月の部分を返します。"""
        ...

    def mapper(self, _, line):
        """各ログ行を解析し、関連する行を抽出して変換します。

        キーと値のペアを次の形式で出力します:

        (2016-01, url0), 1
        (2016-01, url0), 1
        (2016-01, url1), 1
        """
        url = self.extract_url(line)
        period = self.extract_year_month(line)
        yield (period, url), 1

    def reducer(self, key, values):
        """各キーの値を合計します。

        (2016-01, url0), 2
        (2016-01, url1), 1
        """
        yield key, sum(values)
```

### ユースケース: サービスが期限切れのペーストを削除する

期限切れのペーストを削除するには、現在のタイムスタンプよりも古い有効期限のタイムスタンプを持つすべてのエントリを**SQLデータベース**でスキャンするだけです。すべての期限切れエントリはテーブルから削除（または期限切れとしてマーク）されます。

## ステップ4: 設計をスケールする

> 制約を考慮してボトルネックを特定し、対処します。

![Imgur](http://i.imgur.com/4edXG0T.png)

**重要: 初期設計から最終設計に直接飛び込まないでください！**

これを反復的に行うことを述べます: 1) **ベンチマーク/負荷テスト**、2) **ボトルネックのプロファイル**、3) ボトルネックに対処しながら代替案とトレードオフを評価し、4) 繰り返します。初期設計を反復的にスケールする方法のサンプルとして[数百万ユーザーにスケールするシステムをAWSで設計する](../scaling_aws/README.md)を参照してください。

初期設計で遭遇する可能性のあるボトルネックと、それぞれに対処する方法について議論することが重要です。たとえば、複数の**Webサーバー**を持つ**ロードバランサー**を追加することで解決される問題は何ですか？**CDN**？**マスター-スレーブレプリカ**？それぞれの代替案と**トレードオフ**は何ですか？

設計を完成させ、スケーラビリティの問題に対処するためにいくつかのコンポーネントを導入します。内部ロードバランサーは混乱を避けるために表示されていません。

*議論の繰り返しを避けるために*、主要な話題、トレードオフ、および代替案については、以下の[システム設計トピック](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics)を参照してください:

* [DNS](https://github.com/donnemartin/system-design-primer#domain-name-system)
* [CDN](https://github.com/donnemartin/system-design-primer#content-delivery-network)
* [ロードバランサー](https://github.com/donnemartin/system-design-primer#load-balancer)
* [水平スケーリング](https://github.com/donnemartin/system-design-primer#horizontal-scaling)
* [Webサーバー（リバースプロキシ）](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* [APIサーバー（アプリケーション層）](https://github.com/donnemartin/system-design-primer#application-layer)
* [キャッシュ](https://github.com/donnemartin/system-design-primer#cache)
* [リレーショナルデータベース管理システム（RDBMS）](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)
* [SQL書き込みマスター-スレーブフェイルオーバー](https://github.com/donnemartin/system-design-primer#fail-over)
* [マスター-スレーブレプリケーション](https://github.com/donnemartin/system-design-primer#master-slave-replication)
* [一貫性パターン](https://github.com/donnemartin/system-design-primer#consistency-patterns)
* [可用性パターン](https://github.com/donnemartin/system-design-primer#availability-patterns)

**分析データベース**は、Amazon RedshiftやGoogle BigQueryのようなデータウェアハウジングソリューションを使用できます。

**オブジェクトストア**（例えばAmazon S3）は、月間12.7 GBの新しいコンテンツの制約を快適に処理できます。

40の*平均*読み取りリクエスト/秒（ピーク時はそれ以上）に対処するために、人気のあるコンテンツのトラフィックは**メモリキャッシュ**で処理されるべきです。**メモリキャッシュ**は、不均等に分配されたトラフィックやトラフィックのスパイクを処理するのにも役立ちます。**SQL読み取りレプリカ**はキャッシュミスを処理できるはずですが、レプリカが書き込みのレプリケーションで負担をかけられない限りです。

4の*# Pastebin.com（またはBit.ly）の設計

*注: このドキュメントは、重複を避けるために[システム設計トピック](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics)の関連部分に直接リンクしています。一般的な話題、トレードオフ、および代替案についてはリンク先の内容を参照してください。*

**Bit.lyの設計** - これは類似の質問ですが、pastebinは元の未短縮URLの代わりにペースト内容を保存する必要があります。

## ステップ1: ユースケースと制約の概要

> 要件を収集し、問題の範囲を決定します。
> ユースケースと制約を明確にするために質問します。
> 仮定を議論します。

インタビュアーがいない場合、いくつかのユースケースと制約を定義します。

### ユースケース

#### 以下のユースケースのみを扱うように問題の範囲を設定します

* **ユーザー**がテキストブロックを入力し、ランダムに生成されたリンクを取得する
    * 有効期限
        * デフォルト設定では期限切れにならない
        * オプションで時間制限を設定できる
* **ユーザー**がペーストのURLを入力して内容を表示する
* **ユーザー**は匿名
* **サービス**がページの分析を追跡する
    * 月間訪問統計
* **サービス**が期限切れのペーストを削除する
* **サービス**が高可用性を持つ

#### 範囲外

* **ユーザー**がアカウントを登録する
    * **ユーザー**がメールを確認する
* **ユーザー**が登録済みアカウントにログインする
    * **ユーザー**がドキュメントを編集する
* **ユーザー**が可視性を設定する
* **ユーザー**がショートリンクを設定する

### 制約と仮定

#### 仮定を述べる

* トラフィックは均等に分配されていない
* ショートリンクをフォローするのは速くなければならない
* ペーストはテキストのみ
* ページビュー分析はリアルタイムである必要はない
* 1000万人のユーザー
* 月間1000万のペースト書き込み
* 月間1億のペースト読み取り
* 読み取りと書き込みの比率は10:1

#### 使用量を計算する

**インタビュアーにバックオブエンベロープの使用量計算を行うべきか確認してください。**

* ペーストごとのサイズ
    * ペースト内容は1 KB
    * `shortlink` - 7バイト
    * `expiration_length_in_minutes` - 4バイト
    * `created_at` - 5バイト
    * `paste_path` - 255バイト
    * 合計 = 約1.27 KB
* 月間12.7 GBの新しいペースト内容
    * 1.27 KBのペースト * 月間1000万ペースト
    * 3年間で約450 GBの新しいペースト内容
    * 3年間で3億6000万のショートリンク
    * ほとんどが新しいペーストであり、既存のものの更新ではないと仮定
* 平均で1秒あたり4回のペースト書き込み
* 平均で1秒あたり40回の読み取りリクエスト

便利な変換ガイド:

* 月間250万秒
* 1秒あたり1リクエスト = 月間250万リクエスト
* 1秒あたり40リクエスト = 月間1億リクエスト
* 1秒あたり400リクエスト = 月間10億リクエスト

## ステップ2: 高レベルの設計を作成する

> 重要なコンポーネントをすべて含む高レベルの設計を概説します。

![Imgur](http://i.imgur.com/BKsBnmG.png)

## ステップ3: コアコンポーネントを設計する

> 各コアコンポーネントの詳細に入ります。

### ユースケース: ユーザーがテキストブロックを入力し、ランダムに生成されたリンクを取得する

[リレーショナルデータベース](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)を大きなハッシュテーブルとして使用し、生成されたURLをファイルサーバーとペーストファイルを含むパスにマッピングすることができます。

ファイルサーバーを管理する代わりに、Amazon S3のような管理された**オブジェクトストア**や[NoSQLドキュメントストア](https://github.com/donnemartin/system-design-primer#document-store)を使用することができます。

大きなハッシュテーブルとして機能するリレーショナルデータベースの代わりに、[NoSQLキー値ストア](https://github.com/donnemartin/system-design-primer#key-value-store)を使用することもできます。SQLとNoSQLの選択に関する[トレードオフ](https://github.com/donnemartin/system-design-primer#sql-or-nosql)を議論する必要があります。以下の議論ではリレーショナルデータベースアプローチを使用します。

* **クライアント**が**Webサーバー**にペースト作成リクエストを送信します。Webサーバーは[リバースプロキシ](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)として動作します。
* **Webサーバー**がリクエストを**書き込みAPI**サーバーに転送します。
* **書き込みAPI**サーバーは以下を行います:
    * 一意のURLを生成します
        * **SQLデータベース**で重複を確認してURLが一意であることを確認します
        * URLが一意でない場合、別のURLを生成します
        * カスタムURLをサポートする場合、ユーザーが提供したURLを使用します（重複も確認）
    * **SQLデータベース**の`pastes`テーブルに保存します
    * ペーストデータを**オブジェクトストア**に保存します
    * URLを返します

**インタビュアーにどれだけのコードを書くべきか確認してください**。

`pastes`テーブルは以下の構造を持つことができます:

```
shortlink char(7) NOT NULL
expiration_length_in_minutes int NOT NULL
created_at datetime NOT NULL
paste_path varchar(255) NOT NULL
PRIMARY KEY(shortlink)
```

`shortlink`列に基づいて主キーを設定することで、データベースは一意性を強制するための[インデックス](https://github.com/donnemartin/system-design-primer#use-good-indices)を作成します。`created_at`に追加のインデックスを作成して、ルックアップを高速化し（全テーブルをスキャンする代わりにログ時間）、データをメモリに保持します。メモリから1 MBを順次読み取るのに約250マイクロ秒かかりますが、SSDから読み取ると4倍、ディスクから読み取ると80倍長くかかります。<sup><a href=https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know>1</a></sup>

一意のURLを生成するために、以下を行うことができます:

* ユーザーのip_address + タイムスタンプの[**MD5**](https://en.wikipedia.org/wiki/MD5)ハッシュを取ります
    * MD5は128ビットのハッシュ値を生成する広く使用されているハッシュ関数です
    * MD5は均等に分布します
    * 代わりにランダムに生成されたデータのMD5ハッシュを取ることもできます
* MD5ハッシュを[**Base 62**](https://www.kerstner.at/2012/07/shortening-strings-using-base-62-encoding/)でエンコードします
    * Base 62は`[a-zA-Z0-9]`にエンコードされ、URLに適しており、特殊文字のエスケープが不要です
    * 元の入力に対して1つのハッシュ結果しかなく、Base 62は決定論的です（ランダム性は関与しません）
    * Base 64も人気のあるエンコーディングですが、追加の`+`と`/`文字のためURLには問題があります
    * 以下の[Base 62擬似コード](http://stackoverflow.com/questions/742013/how-to-code-a-url-shortener)は、桁数kに対してO(k)時間で実行されます:

```python
def base_encode(num, base=62):
    digits = []
    while num > 0:
      remainder = modulo(num, base)
      digits.push(remainder)
      num = divide(num, base)
    digits = digits.reverse
```

* 出力の最初の7文字を取り、62^7の可能な値を得て、3年間で3億6000万のショートリンクの制約を処理するのに十分です:

```python
url = base_encode(md5(ip_address+timestamp))[:URL_LENGTH]
```

公開[**REST API**](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)を使用します:

```
$ curl -X POST --data '{ "expiration_length_in_minutes": "60", \
    "paste_contents": "Hello World!" }' https://pastebin.com/api/v1/paste
```

レスポンス:

```
{
    "shortlink": "foobar"
}
```

内部通信には[リモートプロシージャコール](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)を使用できます。

### ユースケース: ユーザーがペーストのURLを入力して内容を表示する

* **クライアント**が**Webサーバー**にペースト取得リクエストを送信します
* **Webサーバー**がリクエストを**読み取りAPI**サーバーに転送します
* **読み取りAPI**サーバーは以下を行います:
    * **SQLデータベース**で生成されたURLを確認します
        * URLが**SQLデータベース**にある場合、**オブジェクトストア**からペースト内容を取得します
        * そうでない場合、ユーザーにエラーメッセージを返します

REST API:

```
$ curl https://pastebin.com/api/v1/paste?shortlink=foobar
```

レスポンス:

```
{
    "paste_contents": "Hello World"
    "created_at": "YYYY-MM-DD HH:MM:SS"
    "expiration_length_in_minutes": "60"
}
```

### ユースケース: サービスがページの分析を追跡する

リアルタイム分析が要件でないため、**Webサーバー**のログを**MapReduce**してヒット数を生成するだけで済みます。

**インタビュアーにどれだけのコードを書くべきか確認してください**。

```python
class HitCounts(MRJob):

    def extract_url(self, line):
        """ログ行から生成されたURLを抽出します。"""
        ...

    def extract_year_month(self, line):
        """タイムスタンプから年と月の部分を返します。"""
        ...

    def mapper(self, _, line):
        """各ログ行を解析し、関連する行を抽出して変換します。

        キーと値のペアを次の形式で出力します:

        (2016-01, url0), 1
        (2016-01, url0), 1
        (2016-01, url1), 1
        """
        url = self.extract_url(line)
        period = self.extract_year_month(line)
        yield (period, url), 1

    def reducer(self, key, values):
        """各キーの値を合計します。

        (2016-01, url0), 2
        (2016-01, url1), 1
        """
        yield key, sum(values)
```

### ユースケース: サービスが期限切れのペーストを削除する

期限切れのペーストを削除するには、現在のタイムスタンプよりも古い有効期限のタイムスタンプを持つすべてのエントリを**SQLデータベース**でスキャンするだけです。すべての期限切れエントリはテーブルから削除（または期限切れとしてマーク）されます。

## ステップ4: 設計をスケールする

> 制約を考慮してボトルネックを特定し、対処します。

![Imgur](http://i.imgur.com/4edXG0T.png)

**重要: 初期設計から最終設計に直接飛び込まないでください！**

これを反復的に行うことを述べます: 1) **ベンチマーク/負荷テスト**、2) **ボトルネックのプロファイル**、3) ボトルネックに対処しながら代替案とトレードオフを評価し、4) 繰り返します。初期設計を反復的にスケールする方法のサンプルとして[数百万ユーザーにスケールするシステムをAWSで設計する](../scaling_aws/README.md)を参照してください。

初期設計で遭遇する可能性のあるボトルネックと、それぞれに対処する方法について議論することが重要です。たとえば、複数の**Webサーバー**を持つ**ロードバランサー**を追加することで解決される問題は何ですか？**CDN**？**マスター-スレーブレプリカ**？それぞれの代替案と**トレードオフ**は何ですか？

設計を完成させ、スケーラビリティの問題に対処するためにいくつかのコンポーネントを導入します。内部ロードバランサーは混乱を避けるために表示されていません。

*議論の繰り返しを避けるために*、主要な話題、トレードオフ、および代替案については、以下の[システム設計トピック](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics)を参照してください:

* [DNS](https://github.com/donnemartin/system-design-primer#domain-name-system)
* [CDN](https://github.com/donnemartin/system-design-primer#content-delivery-network)
* [ロードバランサー](https://github.com/donnemartin/system-design-primer#load-balancer)
* [水平スケーリング](https://github.com/donnemartin/system-design-primer#horizontal-scaling)
* [Webサーバー（リバースプロキシ）](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* [APIサーバー（アプリケーション層）](https://github.com/donnemartin/system-design-primer#application-layer)
* [キャッシュ](https://github.com/donnemartin/system-design-primer#cache)
* [リレーショナルデータベース管理システム（RDBMS）](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)
* [SQL書き込みマスター-スレーブフェイルオーバー](https://github.com/donnemartin/system-design-primer#fail-over)
* [マスター-スレーブレプリケーション](https://github.com/donnemartin/system-design-primer#master-slave-replication)
* [一貫性パターン](https://github.com/donnemartin/system-design-primer#consistency-patterns)
* [可用性パターン](https://github.com/donnemartin/system-design-primer#availability-patterns)

**分析データベース**は、Amazon RedshiftやGoogle BigQueryのようなデータウェアハウジングソリューションを使用できます。

**オブジェクトストア**（例えばAmazon S3）は、月間12.7 GBの新しいコンテンツの制約を快適に処理できます。

40の*平均*読み取りリクエスト/秒（ピーク時はそれ以上）に対処するために、人気のあるコンテンツのトラフィックは**メモリキャッシュ**で処理されるべきです。**メモリキャッシュ**は、不均等に分配されたトラフィックやトラフィックのスパイクを処理するのにも役立ちます。**SQL読み取りレプリカ**はキャッシュミスを処理できるはずですが、レプリカが書き込みのレプリケーションで負担をかけられない限りです。

1秒あたり平均4回のペースト書き込み（ピーク時にはそれ以上）は、単一の**SQL書き込みマスター-スレーブ**で対応可能です。それ以外の場合は、追加のSQLスケーリングパターンを採用する必要があります：

* [フェデレーション](https://github.com/donnemartin/system-design-primer#federation)
* [シャーディング](https://github.com/donnemartin/system-design-primer#sharding)
* [非正規化](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQLチューニング](https://github.com/donnemartin/system-design-primer#sql-tuning)

一部のデータを**NoSQLデータベース**に移行することも検討すべきです。

## 追加の話題

> 問題の範囲と残り時間に応じて、さらに掘り下げるトピック。

#### NoSQL

* [キー-バリューストア](https://github.com/donnemartin/system-design-primer#key-value-store)
* [ドキュメントストア](https://github.com/donnemartin/system-design-primer#document-store)
* [ワイドカラムストア](https://github.com/donnemartin/system-design-primer#wide-column-store)
* [グラフデータベース](https://github.com/donnemartin/system-design-primer#graph-database)
* [SQL vs NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)

### キャッシュ

* どこにキャッシュするか
    * [クライアントキャッシュ](https://github.com/donnemartin/system-design-primer#client-caching)
    * [CDNキャッシュ](https://github.com/donnemartin/system-design-primer#cdn-caching)
    * [ウェブサーバーキャッシュ](https://github.com/donnemartin/system-design-primer#web-server-caching)
    * [データベースキャッシュ](https://github.com/donnemartin/system-design-primer#database-caching)
    * [アプリケーションキャッシュ](https://github.com/donnemartin/system-design-primer#application-caching)
* 何をキャッシュするか
    * [データベースクエリレベルでのキャッシュ](https://github.com/donnemartin/system-design-primer#caching-at-the-database-query-level)
    * [オブジェクトレベルでのキャッシュ](https://github.com/donnemartin/system-design-primer#caching-at-the-object-level)
* キャッシュをいつ更新するか
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

* トレードオフを議論する：
    * クライアントとの外部通信 - [RESTに従ったHTTP API](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)
    * 内部通信 - [RPC](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)
* [サービスディスカバリー](https://github.com/donnemartin/system-design-primer#service-discovery)

### セキュリティ

[セキュリティセクション](https://github.com/donnemartin/system-design-primer#security)を参照してください。

### レイテンシーの数値

[プログラマーが知っておくべきレイテンシーの数値](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know)を参照してください。

### 継続的な作業

* ボトルネックが発生したら、システムのベンチマークとモニタリングを続ける
* スケーリングは反復的なプロセスです