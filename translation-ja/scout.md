# Laravel Scout

- [イントロダクション](#introduction)
- [インストール](#installation)
    - [ドライバの事前要件](#driver-prerequisites)
    - [キュー投入](#queueing)
- [設定](#configuration)
    - [モデルインデックスの設定](#configuring-model-indexes)
    - [検索可能データの設定](#configuring-searchable-data)
    - [モデルIDの設定](#configuring-the-model-id)
    - [モデルごとのサーチエンジン設定](#configuring-search-engines-per-model)
    - [ユーザーの識別](#identifying-users)
- [データベース／コレクションエンジン](#database-and-collection-engines)
    - [データベースエンジン](#database-engine)
    - [コレクションエンジン](#collection-engine)
- [インデックス](#indexing)
    - [バッチ取り込み](#batch-import)
    - [レコード追加](#adding-records)
    - [レコード更新](#updating-records)
    - [レコード削除](#removing-records)
    - [インデックスの一時停止](#pausing-indexing)
    - [条件付き検索可能モデルインスタンス](#conditionally-searchable-model-instances)
- [検索](#searching)
    - [Where節](#where-clauses)
    - [ペジネーション](#pagination)
    - [ソフトデリート](#soft-deleting)
    - [エンジンの検索のカスタマイズ](#customizing-engine-searches)
- [カスタムエンジン](#custom-engines)
- [ビルダマクロ](#builder-macros)

<a name="introduction"></a>
## イントロダクション

[Laravel Scout](https://github.com/laravel/scout)（Scout、斥候）は、[Eloquentモデル](/docs/{{version}}/eloquent)へ、シンプルなドライバベースのフルテキストサーチを提供します。モデルオブサーバを使い、Scoutは検索インデックスを自動的にEloquentレコードと同期します。

現在、Scoutは[Algolia](https://www.algolia.com/), [MeiliSearch](https://www.meilisearch.com), MySQL／PostgreSQL (`database`) ドライバを用意しています。さらに、Scoutはローカル開発用途に設計された、外部依存やサードパーティサービスを必要としない「コレクション」ドライバも用意しています。加えて、カスタムドライバの作成も簡単で、独自の検索実装でScoutを自由に拡張可能です。

<a name="installation"></a>
## インストール

最初に、Composerパッケージマネージャを使い、Scoutをインストールします。

```shell
composer require laravel/scout
```

Scoutをインストールした後、`vendor:publish` Artisanコマンドを実行してScout設定ファイルをリソース公開する必要があります。このコマンドは、`scout.php`設定ファイルをアプリケーションの`config`ディレクトリへリソース公開します。

```shell
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

最後に、検索可能にしたいモデルに`Laravel\Scout\Searchable`トレイトを追加します。このトレイトは、モデルを検索ドライバと自動的に同期させるモデルオブザーバを登録します。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\Searchable;

    class Post extends Model
    {
        use Searchable;
    }

<a name="driver-prerequisites"></a>
### ドライバの事前要件

<a name="algolia"></a>
#### Algolia

Algoliaドライバを使用する場合、Algolia `id`と`secret`接続情報を`config/scout.php`設定ファイルで設定する必要があります。接続情報を設定し終えたら、Algolia PHP SDKをComposerパッケージマネージャで、インストールする必要があります。

```shell
composer require algolia/algoliasearch-client-php
```

<a name="meilisearch"></a>
#### MeiliSearch

[MeiliSearch](https://www.meilisearch.com)は、非常に高速なオープンソースの検索エンジンです。ローカルマシンにMeiliSearchをインストールする方法がわからない場合は、Laravelの公式サポートのDocker開発環境である[Laravel Sail](/docs/{{version}}/sail#meilisearch)を利用できます。

Meilisearchドライバを使用する場合は、Composerパッケージマネージャを使用して、MeiliSearch PHP SDKをインストールする必要があります。

```shell
composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle
```

次に、アプリケーションの`.env`ファイル内の`SCOUT_DRIVER`環境変数とMeiliSearch`host`と`key`認証情報を設定します。

```ini
SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=masterKey
```

MeiliSearchの詳細については、[MeiliSearchのドキュメント](https://docs.meilisearch.com/learn/getting_started/quick_start.html)を参照してください。

さらに、[MeiliSearchのバイナリ互換のドキュメント](https://github.com/meilisearch/meilisearch-php#-compatibility-with-meilisearch)を見て、自分が使っているMeiliSearchのバイナリバージョンと互換性のあるバージョンの`meilisearch/meilisearch-php`をインストールしてください。

> **Warning**
> MeiliSearchを利用しているアプリケーションのScoutをアップグレードする際には、常にMeiliSearchサービス自体に[追加の破壊的な変更](https://github.com/meilisearch/MeiliSearch/releases)がないか確認する必要があります。

<a name="queueing"></a>
### キュー投入

厳密にはScoutを使用する必要はありませんが、ライブラリを使用する前に、[キュードライバ](/docs/{{version}}/queues)の設定を強く考慮する必要があります。キューワーカを実行すると、Scoutはモデル情報を検索インデックスに同期するすべての操作をキューに入れることができ、アプリケーションのWebインターフェイスのレスポンス時間が大幅に短縮されます。

キュードライバを設定したら、`config/scout.php`設定ファイルの`queue`オプションの値を`true`に設定します。

    'queue' => true,

`queue`オプションを`false`に設定している場合でも、AlgoliaやMeilisearchなど、一部のScoutドライバは常に非同期でレコードをインデックスしていることを覚えておくことが重要です。つまり、Laravelアプリケーション内でインデックス操作が完了しても、検索エンジン自体には新しいレコードや更新されたレコードがすぐに反映されない可能性があります。

Scoutジョブで利用する接続とキューを指定するには、`queue`設定オプションを配列として定義します。

    'queue' => [
        'connection' => 'redis',
        'queue' => 'scout'
    ],

<a name="configuration"></a>
## 設定

<a name="configuring-model-indexes"></a>
### モデルインデックスの設定

各Eloquentモデルは、検索可能レコードすべてを含む、指定された検索「インデックス」と同期されます。言い換えれば、各インデックスはMySQLテーブルのようなものであると、考えられます。デフォルトで、各モデルはそのモデルの典型的な「テーブル」名に一致するインデックスへ保存されます。通常、モデルの複数形ですが、モデルの`searchableAs`メソッドをオーバーライドすることで、このモデルのインデックスを自由にカスタマイズ可能です。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\Searchable;

    class Post extends Model
    {
        use Searchable;

        /**
         * モデルに関連付けられているインデックスの名前を取得
         *
         * @return string
         */
        public function searchableAs()
        {
            return 'posts_index';
        }
    }

<a name="configuring-searchable-data"></a>
### 検索可能データの設定

デフォルトでは、指定されたモデルの`toArray`形態全体が、検索インデックスへ保存されます。検索インデックスと同期するデータをカスタマイズしたい場合は、そのモデルの`toSearchableArray`メソッドをオーバーライドできます。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\Searchable;

    class Post extends Model
    {
        use Searchable;

        /**
         * モデルのインデックス可能なデータ配列の取得
         *
         * @return array
         */
        public function toSearchableArray()
        {
            $array = $this->toArray();

            // データ配列をカスタマイズ

            return $array;
        }
    }

<a name="configuring-the-model-id"></a>
### モデルIDの設定

デフォルトでは、Scoutはモデルの主キーを、検索インデックスに保存されているモデルの一意のID／キーとして使用します。この動作をカスタマイズする必要がある場合は、モデルの`getScoutKey`メソッドと`getScoutKeyName`メソッドをオーバーライドできます。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\Searchable;

    class User extends Model
    {
        use Searchable;

        /**
         * モデルのインデックスに使用する値の取得
         *
         * @return mixed
         */
        public function getScoutKey()
        {
            return $this->email;
        }

        /**
         * モデルのインデックスに使用するキー名の取得
         *
         * @return mixed
         */
        public function getScoutKeyName()
        {
            return 'email';
        }
    }

<a name="configuring-search-engines-per-model"></a>
### モデルごとのサーチエンジン設定

検索時、Scoutはアプリケーションの`scout`設定ファイルで指定したデフォルト検索エンジンを通常使用します。しかし、特定モデルの検索エンジンを変更したい場合は、そのモデルの`searchableUsing`メソッドをオーバーライドしてください。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\EngineManager;
    use Laravel\Scout\Searchable;

    class User extends Model
    {
        use Searchable;

        /**
         * Get the engine used to index the model.
         *
         * @return \Laravel\Scout\Engines\Engine
         */
        public function searchableUsing()
        {
            return app(EngineManager::class)->engine('meilisearch');
        }
    }

<a name="identifying-users"></a>
### ユーザーの識別

Scoutを使用すると、[Algolia](https://algolia.com)を使用するときにユーザーを自動識別することもできます。認証済みユーザーを検索操作に関連付けると、Algoliaのダッシュボード内で検索分析を表示するときに役立つ場合があります。アプリケーションの`.env`ファイルで`SCOUT_IDENTIFY`環境変数を`true`として定義することにより、ユーザー識別を有効にできます。

```ini
SCOUT_IDENTIFY=true
```

この機能を有効にすると、リクエストのIPアドレスと認証済みユーザーのプライマリ識別子もAlgoliaに渡されるため、これらのデータはそのユーザーが行った検索リクエストへ関連付けられます。

<a name="database-and-collection-engines"></a>
## データベース／コレクションエンジン

<a name="database-engine"></a>
### データベースエンジン

> **Warning**
> 現在、データベースエンジンは、MySQLとPostgreSQLをサポートしています。

中規模のデータベースとやり取りするしたり、作業負荷が軽いアプリケーションでは、Scoutの「データベース」エンジンで始めるのが便利でしょう。データベースエンジンは、既存のデータベースから結果をフィルタリングする際に、「where like」句と全文インデックスを使用して、クエリの検索結果を決定します。

データベースエンジンを使うには、`SCOUT_DRIVER`環境変数の値を`database`に設定するか、アプリケーションの`scout`設定ファイルに直接`database`ドライバを指定してください。

```ini
SCOUT_DRIVER=database
```

データベースエンジンを好みのドライバに指定したら、[検索可能なデータの設定](#configuring-searchable-data)を行う必要があります。次に、モデルに対して[検索クエリの実行](#searching)を開始します。データベースエンジンを使用する場合、AlgoliaやMeiliSearchのように、検索エンジンのインデックス作成は必要ありません。

#### データベース検索戦略のカスタマイズ

デフォルトでは、データベースエンジンは、[検索可能](#configuring-searchable-data)として設定したすべてのモデル属性に対して、"WHERE LIKE"クエリを実行します。しかし、この方法では状況により、パフォーマンス低下を招くことがあります。そこで、データベースエンジンの検索戦略を設定することで、指定した一部のカラムでは全文検索クエリを利用し、あるいは文字列全体を検索（`%example%`）するのではなく、前方一致で検索する（`example%`）"WHERE LIKE"制約のみを利用できます。

こうした振る舞いを定義するには、モデルの`toSearchableArray`メソッドでPHP属性を割り付けてください。追加の検索戦略動作を割り当てていないカラムには、デフォルトの"WHERE LIKE"戦略を使い続けます。

```php
use Laravel\Scout\Attributes\SearchUsingFullText;
use Laravel\Scout\Attributes\SearchUsingPrefix;

/**
 * モデルに対するインデックス可能なデータの配列を取得
 *
 * @return array
 */
#[SearchUsingPrefix(['id', 'email'])]
#[SearchUsingFullText(['bio'])]
public function toSearchableArray()
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        'bio' => $this->bio,
    ];
}
```

> **Warning**
> あるカラムへ、フルテキストクエリ制約の使用を指定する前に、そのカラムに[フルテキストインデックス](/docs/{{version}}/migrations#available-index-types)を割り当て済みであることを確認してください。

<a name="collection-engine"></a>
### コレクションエンジン

ローカル開発時には、AlgoliaやMeiliSearchの検索エンジンを自由に使用することができますが、「コレクション（collection）」エンジンでスタートした方が便利な場合もあります。コレクション・エンジンは、既存データベースからの結果に対して、"where"節とコレクション・フィルタリングを用いて、クエリに該当する検索結果を決定します。このエンジンを使用する場合、Searchableモデルをインデックス化する必要はなく、シンプルにローカル・データベースから検索します。

コレクションエンジンを使用するには，環境変数`SCOUT_DRIVER`の値を`collection`に設定するか，アプリケーションの`scout`設定ファイルで`collection`ドライバを直接指定します。

```ini
SCOUT_DRIVER=collection
```

コレクションドライバを使用ドライバに指定したら、モデルに対して[検索クエリの実行](#searching)を開始できます。コレクションエンジンを使用する場合、AlgoliaやMeiliSearchのインデックスのシードに必要なインデックス作成などの検索エンジンのインデックス作成は不要です。

#### データベースエンジンとの違い

一見すると、「データベース」エンジンと「コレクション」エンジンはかなり似ています。どちらも、あなたのデータベースと直接やりとりして、検索結果を取得します。しかし、コレクションエンジンはフルテキストインデックスや`LIKE`句を利用して一致レコードを探し出しません。その代わりに、可能性があるすべてのレコードを取得し、Laravelの`Str::is`ヘルパを使い、モデルの属性値内に検索文字列が存在しているか判断します。

コレクションエンジンは、Laravelがサポートする（SQLiteやSQL Serverを含む）すべてのリレーショナルデータベースで動作するため、最も使いやすい検索エンジンですが、Scoutのデータベースエンジンに比べると効率は落ちます。

<a name="indexing"></a>
## インデックス

<a name="batch-import"></a>
### バッチ取り込み

Scoutを既存のプロジェクトにインストールする場合は、インデックスへインポートする必要のあるデータベースレコードがすでに存在している可能性があります。Scoutは、既存のすべてのレコードを検索インデックスにインポートするために使用できる`scout:import` Artisanコマンドを提供しています。

```shell
php artisan scout:import "App\Models\Post"
```

`flush`コマンドは、検索インデックスからモデルの全レコードを削除するために使用します。

```shell
php artisan scout:flush "App\Models\Post"
```

<a name="modifying-the-import-query"></a>
#### インポートクエリの変更

バッチインポートで全モデルを取得するために使用されるクエリを変更する場合は、モデルに`makeAllSearchableUsing`メソッドを定義してください。これはモデルをインポートする前に、必要になる可能性のあるイエガーリレーションの読み込みを追加するのに最適な場所です。

    /**
     * 全モデルを検索可能にするときの、モデル取得に使用するクエリを変更
     *
     * @param  \Illuminate\Database\Eloquent\Builder  $query
     * @return \Illuminate\Database\Eloquent\Builder
     */
    protected function makeAllSearchableUsing($query)
    {
        return $query->with('author');
    }

<a name="adding-records"></a>
### レコード追加

モデルに`Laravel\Scout\Searchable`トレイトを追加したら、モデルインスタンスを`保存`または`作成`するだけで、検索インデックスに自動的に追加されます。[キューを使用](#queueing)するようにScoutを設定した場合、この操作はキューワーカによってバックグラウンドで実行されます。

    use App\Models\Order;

    $order = new Order;

    // ...

    $order->save();

<a name="adding-records-via-query"></a>
#### クエリによるレコード追加

Eloquentクエリを介してモデルのコレクションを検索インデックスに追加する場合は、`searchable`メソッドをEloquentクエリにチェーンできます。`searchable`メソッドはクエリの[結果をチャンク](/docs/{{version}}/eloquent#chunking-results)し、レコードを検索インデックスに追加します。繰り返しますが、キューを使用するようにScoutを設定した場合、すべてのチャンクはキューワーカによってバックグラウンドでインポートされます。

    use App\Models\Order;

    Order::where('price', '>', 100)->searchable();

Eloquentリレーションインスタンスで `searchable`メソッドを呼び出すこともできます。

    $user->orders()->searchable();

または、メモリ内にEloquentモデルのコレクションが既にある場合は、コレクションインスタンスで`searchable`メソッドを呼び出して、モデルインスタンスを対応するインデックスに追加できます。

    $orders->searchable();

> **Note**
> `searchable`メソッドは、「アップサート（upsert）」操作と考えるられます。つまり、モデルレコードがすでにインデックスに含まれている場合は、更新され、検索インデックスに存在しない場合は追加されます。

<a name="updating-records"></a>
### レコード更新

検索可能モデルを更新するには、モデルインスタンスのプロパティを更新し、`save`でモデルをデータベースへ保存します。Scoutは自動的に変更を検索インデックスへ保存します。

    use App\Models\Order;

    $order = Order::find(1);

    // 注文を更新…

    $order->save();

Eloquentクエリインスタンスで`searchable`メソッドを呼び出して、モデルのコレクションを更新することもできます。モデルが検索インデックスに存在しない場合は作成されます。

    Order::where('price', '>', 100)->searchable();

リレーションシップ内のすべてのモデルの検索インデックスレコードを更新する場合は、リレーションシップインスタンスで`searchable`を呼び出すことができます。

    $user->orders()->searchable();

または、メモリ内にEloquentモデルのコレクションが既にある場合は、コレクションインスタンスで`searchable`メソッドを呼び出して、対応するインデックスのモデルインスタンスを更新できます。

    $orders->searchable();

<a name="removing-records"></a>
### レコード削除

インデックスからレコードを削除するには、データベースからモデルを`delete`するだけです。これは、[ソフトデリート](/docs/{{version}}/eloquent#soft-deleting)モデルを使用している場合でも実行できます。

    use App\Models\Order;

    $order = Order::find(1);

    $order->delete();

レコードを削除する前にモデルを取得したくない場合は、Eloquentクエリインスタンスで`unsearchable`メソッドを使用できます。

    Order::where('price', '>', 100)->unsearchable();

リレーション内のすべてのモデルの検索インデックスレコードを削除する場合は、リレーションインスタンスで`unsearchable`を呼び出してください。

    $user->orders()->unsearchable();

または、メモリ内にEloquentモデルのコレクションが既にある場合は、コレクションインスタンスで`unsearchable`メソッドを呼び出して、対応するインデックスからモデルインスタンスを削除できます。

    $orders->unsearchable();

<a name="pausing-indexing"></a>
### インデックスの一時停止

モデルデータを検索インデックスに同期せずに、モデルに対してEloquent操作のバッチを実行する必要がある場合があります。これは、`withoutSyncingToSearch`メソッドを使用して行うことができます。このメソッドは、すぐに実行される単一のクロージャを引数に取ります。クロージャ内で発行するモデル操作は、モデルのインデックスに同期されません。

    use App\Models\Order;

    Order::withoutSyncingToSearch(function () {
        // モデルアクションを実行
    });

<a name="conditionally-searchable-model-instances"></a>
### 条件付き検索可能モデルインスタンス

特定の条件下でのみ、モデルを検索可能にする必要がある場合も起きるでしょう。たとえば、`App\Models\Post`モデルが、"draft"か"published"の２つのうち、どちらか１つの状態を取ると想像してください。「公開済み:published」のポストのみ検索可能にする必要があります。これを実現するには、モデルに`shouldBeSearchable`メソッドを定義してください。

    /**
     * モデルを検索可能にする判定
     *
     * @return bool
     */
    public function shouldBeSearchable()
    {
        return $this->isPublished();
    }

`shouldBeSearchable`メソッドは、`save`および`create`メソッド、クエリ、またはリレーションを通してモデルを操作する場合にのみ適用されます。`searchable`メソッドを使用してモデルまたはコレクションを直接検索可能にすると、`shouldBeSearchable`メソッドの結果が上書きされます。

> **Warning**
> 検索可能なデータは常にデータベースへ保存されるため、`shouldBeSearchable`メソッドはScoutの「データベース」エンジンを使用する際には適用されません。データベースエンジン使用時に同様の動作をさせるには、代わりに[WHERE句](#where-clauses)を使用する必要があります。

<a name="searching"></a>
## 検索

`search`メソッドにより、モデルの検索を開始しましょう。`search`メソッドはモデルを検索するために使用する文字列だけを引数に指定します。`get`メソッドを検索クエリにチェーンし、指定した検索クエリに一致するEloquentモデルを取得できます。

    use App\Models\Order;

    $orders = Order::search('Star Trek')->get();

Scoutの検索ではEloquentモデルのコレクションが返されるため、ルートやコントローラから直接結果を返せば、自動的にJSONへ変換されます。

    use App\Models\Order;
    use Illuminate\Http\Request;

    Route::get('/search', function (Request $request) {
        return Order::search($request->search)->get();
    });

Eloquentモデルへ変換する前に素の検索結果を取得したい場合は、`raw`メソッドを使用できます。

    $orders = Order::search('Star Trek')->raw();

<a name="custom-indexes"></a>
#### カスタムインデックス

検索クエリは通常、モデルの[`searchableAs`](#configuring-model-indexes)メソッドで指定するインデックスに対して実行されます。ただし、`within`メソッドを使用して、代わりに検索する必要があるカスタムインデックスを指定できます。

    $orders = Order::search('Star Trek')
        ->within('tv_shows_popularity_desc')
        ->get();

<a name="where-clauses"></a>
### Where節

Scoutを使用すると、検索クエリに単純な「where」節を追加できます。現在、これらの節は基本的な数値の同等性チェックのみをサポートしており、主に所有者IDによる検索クエリのスコープに役立ちます。

    use App\Models\Order;

    $orders = Order::search('Star Trek')->where('user_id', 1)->get();

`whereIn`メソッドを使用すると、指定された値の集合に対して結果を制約できます。

    $orders = Order::search('Star Trek')->whereIn(
        'status', ['paid', 'open']
    )->get();

検索インデックスはリレーショナルデータベースではないため、より高度な"where"節は現在サポートしていません。

<a name="pagination"></a>
### ペジネーション

モデルのコレクションを取得することに加えて、`paginate`メソッドを使用して検索結果をページ分割することができます。このメソッドは、[従来のEloquentクエリをペジネーションする](/docs/{{version}}/pagination)場合と同じように、`Illuminate\Pagination\LengthAwarePaginator`インスタンスを返します。

    use App\Models\Order;

    $orders = Order::search('Star Trek')->paginate();

`paginate`メソッドの第１引数として、各ページごとに取得したいモデル数を指定します。

    $orders = Order::search('Star Trek')->paginate(15);

結果が取得できたら、通常のEloquentクエリのペジネーションと同様に、結果を表示し、[Blade](/docs/{{version}}/blade)を使用してページリンクをレンダできます。

```html
<div class="container">
    @foreach ($orders as $order)
        {{ $order->price }}
    @endforeach
</div>

{{ $orders->links() }}
```

もちろん、ペジネーションの結果をJSONとして取得したい場合は、ルートまたはコントローラから直接ペジネータインスタンスを返すことができます。

    use App\Models\Order;
    use Illuminate\Http\Request;

    Route::get('/orders', function (Request $request) {
        return Order::search($request->input('query'))->paginate(15);
    });

> **Warning**
> 検索エンジンはEloquentモデルのグローバルスコープ定義を認識しないため、Scoutのペジネーションを利用するアプリケーションではグローバルスコープを使うべきでありません。それでも、Scoutにより検索する場合は、グローバルスコープの制約を再作成する必要があります。

<a name="soft-deleting"></a>
### ソフトデリート

インデックス付きのモデルが[ソフトデリート](/docs/{{version}}/eloquent#soft-deleting)され、ソフトデリート済みのモデルをサーチする必要がある場合、`config/scout.php`設定ファイルの`soft_delete`オプションを`true`に設定してください。

    'soft_delete' => true,

この設定オプションを`true`にすると、Scoutは検索インデックスからソフトデリートされたモデルを削除しません。代わりに、インデックスされたレコードへ、隠し`__soft_deleted`属性をセットします。これにより、検索時にソフトデリート済みレコードを取得するために、`withTrashed`や`onlyTrashed`メソッドがつかえます。

    use App\Models\Order;

    // 結果の取得時に、削除済みレコードも含める
    $orders = Order::search('Star Trek')->withTrashed()->get();

    // 結果の取得時に、削除済みレコードのみを対象とする
    $orders = Order::search('Star Trek')->onlyTrashed()->get();

> **Note**
> ソフトデリートされたモデルが、`forceDelete`により完全に削除されると、Scoutは自動的に検索インデックスから削除します。

<a name="customizing-engine-searches"></a>
### エンジンの検索のカスタマイズ

エンジンの検索動作の高度なカスタマイズを実行する必要がある場合は、 `search`メソッドの２番目の引数にクロージャを渡せます。たとえば、このコールバックを使用して、検索クエリがAlgoliaに渡される前に、地理的位置データを検索オプションに追加できます。

    use Algolia\AlgoliaSearch\SearchIndex;
    use App\Models\Order;

    Order::search(
        'Star Trek',
        function (SearchIndex $algolia, string $query, array $options) {
            $options['body']['query']['bool']['filter']['geo_distance'] = [
                'distance' => '1000km',
                'location' => ['lat' => 36, 'lon' => 111],
            ];

            return $algolia->search($query, $options);
        }
    )->get();

<a name="customizing-the-eloquent-results-query"></a>
#### Customizing The Eloquent Results Query

Scoutがアプリケーションの検索エンジンからマッチするEloquentモデルのリストを取得した後、Eloquentを使用して主キーでマッチするすべてのモデルを取得しようとします。このクエリは`query`メソッドを呼び出し、カスタマイズできます。`query`メソッドは、Eloquentクエリビルダのインスタンスを引数とするクロージャを受け取ります。

```php
use App\Models\Order;

$orders = Order::search('Star Trek')
    ->query(fn ($query) => $query->with('invoices'))
    ->get();
```

このコールバックは、関連モデルがアプリケーションの検索エンジンからあらかじめ取得された後に呼び出されるので、`query`メソッドを結果の「フィルタリング」に使用しないでください。代わりに、[Scout WHERE句](#where-clauses)を使用してください。

<a name="custom-engines"></a>
## カスタムエンジン

<a name="writing-the-engine"></a>
#### エンジンのプログラミング

組み込みのScout検索エンジンがニーズに合わない場合、独自のカスタムエンジンを書き、Scoutへ登録してください。エンジンは、`Laravel\Scout\Engines\Engine`抽象クラスを拡張してください。この抽象クラスは、カスタムエンジンが実装する必要のある、８つのメソッドを持っています。

    use Laravel\Scout\Builder;

    abstract public function update($models);
    abstract public function delete($models);
    abstract public function search(Builder $builder);
    abstract public function paginate(Builder $builder, $perPage, $page);
    abstract public function mapIds($results);
    abstract public function map(Builder $builder, $results, $model);
    abstract public function getTotalCount($results);
    abstract public function flush($model);

これらのメソッドの実装をレビューするために、`Laravel\Scout\Engines\AlgoliaEngine`クラスが役に立つでしょう。このクラスは独自エンジンで、各メソッドをどのように実装すればよいかの、良い取り掛かりになるでしょう。

<a name="registering-the-engine"></a>
#### エンジンの登録

カスタムエンジンを作成したら、Scoutエンジンマネージャの`extend`メソッドを使用してScoutへ登録します。Scoutのエンジンマネージャは、Laravelサービスコンテナが依存解決できます。`App\Providers\AppServiceProvider`クラスの`boot`メソッドまたはアプリケーションが使用している他のサービスプロバイダから`extend`メソッドを呼び出せます。

    use App\ScoutExtensions\MySqlSearchEngine
    use Laravel\Scout\EngineManager;

    /**
     * 全アプリケーションサービスの初期起動処理
     *
     * @return void
     */
    public function boot()
    {
        resolve(EngineManager::class)->extend('mysql', function () {
            return new MySqlSearchEngine;
        });
    }

エンジンを登録したら、アプリケーションの`config/scout.php`設定ファイルでデフォルトのスカウト`driver`として指定できます。

    'driver' => 'mysql',

<a name="builder-macros"></a>
## ビルダマクロ

カスタムのScout検索ビルダメソッドを定義する場合は、`Laravel\Scout\Builder`クラスで`macro`メソッドが使用できます。通常、「マクロ」は[サービスプロバイダ](/docs/{{version}}/provider)の`boot`メソッド内で定義する必要があります。

    use Illuminate\Support\Facades\Response;
    use Illuminate\Support\ServiceProvider;
    use Laravel\Scout\Builder;

    /**
     * 全アプリケーションサービスの初期起動処理
     *
     * @return void
     */
    public function boot()
    {
        Builder::macro('count', function () {
            return $this->engine->getTotalCount(
                $this->engine()->search($this)
            );
        });
    }

`macro`関数は、最初の引数にマクロ名、２番目の引数にクロージャを取ります。マクロのクロージャは、`Laravel\Scout\Builder`実装からマクロ名を呼び出すときに実行されます。

    use App\Models\Order;

    Order::search('Star Trek')->count();
